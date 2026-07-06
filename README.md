## Features

### RAM Overlay for System Directories

* Critical system directories are overlaid in RAM using OverlayFS for maximum performance:
  * `/etc` - System configuration
  * `/var/log` - System logs
* **Performance:** 9.3x faster IOPS (877K vs 94.6K), 7.2x lower latency (0.52ms vs 3.74ms)
* **Efficiency:** Uses only ~200MB RAM for typical workloads (~1-2% of 16GB)
* **Automatic:** Activates on login, syncs periodically during the session (every 5 min by default) and on logout
* Easily extensible via the `OVERLAY_DIRS` array

### Tmpfs Mounts for Temporary Data

High-speed RAM storage for temporary files (automatically cleaned on logout):
* `/tmp` (5 GB) and `/var/tmp` (1 GB)
* `/var/cache` (2 GB) and `/home/$USER/.cache` (2 GB per user)

### Persistent Caches (Always on Disk)

Essential caches bind-mounted to `/persist` to survive reboots:
* System: `/var/cache/pacman`
* User: `nvidia`, `mesa_shader_cache`, `mesa_shader_cache_db`
* Automatically migrated on first run

### Excluded from Overlay (Remain on Disk)

* `/home` - User data (too large/important)
* `/var/lib` - Databases, flatpak packages (too large)
* `/opt`, `/usr/local` - Large applications
* `/proc`, `/sys`, `/dev`, `/run` - Virtual filesystems
* `/mnt`, `/media`, `/boot` - Mount points

Separately, within an overlaid directory, `/var/log/journal` is excluded from both sync and the RAM pre-population step (`EXCLUDED_FROM_SYNC`) — it's high-churn, journald manages its own persistence, and copying it into RAM up front would burn time and memory for data that's never written back to disk anyway.

### Intelligent Cleanup & Safety

* **Periodic cleanup:** Removes stale files >5 minutes old every 30 seconds
* **Periodic sync:** Incrementally syncs RAM changes back to disk every 5 minutes while a session is active (`SYNC_INTERVAL`, disable with `0`), in addition to the sync at logout — bounds how much a crash or power loss mid-session could lose
* **File protection:** Skips in-use files (via `lsof`, `fuser`, or `/proc`)
* **Graceful shutdown:** Catches SIGTERM/SIGINT/SIGHUP *and* unexpected exits, syncs before exit
* **Comprehensive logging:** All operations tracked in `/var/log/ramoverlay.log`

### Optional Performance Boost

Preloads [mimalloc](https://github.com/microsoft/mimalloc) for `rsync`, `find`, and `inotifywait` if available, reducing memory fragmentation during sync operations and long-lived event watching.

---

## Requirements

* **Commands:** `rsync` (falls back to `mv`), and `lsof` or `fuser` (falls back to `/proc` scanning) - none are hard requirements but `rsync` is strongly recommended for safe syncing
* **RAM:** 16GB+ recommended (typical usage: 200MB-2GB)
* **Filesystem:** Supports OverlayFS (ext4, btrfs, xfs, f2fs)
* **Optional:** `libmimalloc.so` for faster syncs
* **Optional:** Kernel 6.3+ for the tmpfs `noswap` mount option — tried first and falls back cleanly on older kernels, so it's not a hard requirement

---

## Installation

```bash
sudo cp ephemeral-overlay /bin/
sudo chmod 755 /bin/ephemeral-overlay
```

**On rc.local systems**, add to `/etc/rc.local`:

```bash
# Start ephemeral overlay daemon
if [ -x /bin/ephemeral-overlay ]; then
    ( sleep 20; setsid nohup /bin/ephemeral-overlay >> /var/log/ramoverlay.log 2>&1 ) &
fi
```

Registering and enabling it (compiling the service database, adding it to whatever bundle your other longruns are in) is the part that varies by Artix's `s6-rc` frontend version — use the same `s6 set` / `s6 live install` workflow you've already got working for other services rather than any specific commands here, and confirm with `s6-rc-db list` (or your usual check) that it's actually running before trusting it.

**Note:** Runs as a daemon, automatically managing overlay lifecycle based on user sessions.

---

## Configuration

Edit `/bin/ephemeral-overlay`:

**Directories to overlay:**
```bash
OVERLAY_DIRS=(
    "/etc"
    "/var/log"
    # "/opt"          # Add more as needed
)
```

**Persistent system caches:**
```bash
BIND_MOUNTED_VAR_CACHE=(pacman)
```

**Persistent user caches:**
```bash
BIND_MOUNTED_USER_CACHE=(nvidia mesa_shader_cache mesa_shader_cache_db)
```

**Tmpfs sizes:**
```bash
TMP_SIZE="5G"
VAR_TMP_SIZE="1G"
VAR_CACHE_SIZE="2G"
USER_CACHE_SIZE="2G"
```

**Cleanup and sync settings:**
```bash
CLEAN_INTERVAL=30        # Check every 30 seconds
STALE_MINUTES=5          # Delete files older than 5 minutes
SYNC_INTERVAL=300        # Sync RAM -> disk every 5 min while active; 0 to disable
```

---

## Monitoring

```bash
# View active overlays
findmnt -t overlay

# Check RAM usage
df -h /ram_overlay

# See what's in RAM
sudo du -sh /ram_overlay/upper/*

# Monitor activity
tail -f /var/log/ramoverlay.log
```

**Typical RAM usage:**
* Light: 200-500 MB (~1-2% of 16GB)
* Heavy: 1-2 GB (~5-10% of 16GB)
* Allocated: 90% of total RAM (only uses what's needed)

---

## How It Works

1. **Wait:** Daemon waits for user login
2. **Activate:** On login, creates RAM overlay and tmpfs mounts
3. **Operate:** All writes to overlaid directories go to RAM (9.3x faster)
4. **Cleanup:** Daemon removes stale temp files every 30 seconds
5. **Sync:** Incrementally syncs changed files back to disk every 5 minutes while the session is active, then fully on logout
6. **Sleep:** Unmounts everything, frees RAM, waits for next login

---

## Performance (vs SATA SSD)

| Metric | SATA Disk | RAM Overlay | Improvement |
|--------|-----------|-------------|-------------|
| Random Write IOPS | 94,600 | 877,000 | **9.3x faster** |
| Write Latency | 3.74ms | 0.52ms | **7.2x lower** |
| Sequential Write | 530 MB/s | 2,128 MB/s | **4x faster** |
| RAM Overhead | 0 | ~200MB | Minimal |

*Benchmarked on Samsung 870 EVO SATA SSD.* The RAM-vs-SSD comparison itself is a hardware characteristic, unaffected by anything above.

---

## Safety Features

* **Best-effort persistence:** Changes sync to disk periodically during the session, and on logout/shutdown; sync errors are detected and logged, but there is no retry logic or post-sync verification pass
* **Signal handling:** Gracefully handles SIGTERM, SIGINT, and SIGHUP, and also traps `EXIT` directly so an unexpected termination (not just a clean signal) still triggers a sync attempt before the process actually dies. Sleeps in the main loop are broken into 1-second increments rather than one long `sleep N` — bash only checks for a pending trap between commands, so a signal arriving mid-sleep would otherwise sit queued for up to `CLEAN_INTERVAL` (30s by default) before shutdown actually started, which is long enough that a process supervisor with a shorter kill-escalation timeout could SIGKILL it first and skip the sync entirely
* **File-in-use detection:** Never deletes files that are currently open
* **Rsync verification:** Checks rsync exit codes (captured independently of `tee`) and logs sync failures
* **Protected directories:** User data in `/home` is never overlaid
* **OverlayFS semantics:** Per-file writes are atomic via OverlayFS; however, the sync process (unmount → rsync → remount) is not crash-safe - a power loss mid-sync may leave the filesystem partially written. The overlay mount deliberately does *not* use the `volatile` option — it looked like a good fit (upper is tmpfs, never durable across a reboot anyway, so overlayfs's own sync/fsync bookkeeping on it seemed redundant) but testing showed it writes a marker into workdir that makes the kernel refuse every subsequent mount using that workdir, with or without `volatile`, until it's wiped — which breaks the remount this daemon does on every single sync
* **Fallback mechanisms:** Multiple methods for file-in-use detection
