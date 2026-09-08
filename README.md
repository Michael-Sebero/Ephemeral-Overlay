## Summary
Ephemeral Overlay speeds up specified directories by moving them into RAM, reducing disk I/O and increasing system responsiveness.

## Features

### RAM Overlay for System Directories

* Critical system directories run from RAM through OverlayFS:
  * `/etc` - System configuration
  * `/var/log` - System logs
* **Performance:** 9.3x faster IOPS (877K vs 94.6K), 7.2x lower latency (0.52ms vs 3.74ms)
* **Efficiency:** Around 200MB RAM for typical workloads, about 1-2% of 16GB
* **Automatic:** Activates on login, syncs periodically during the session and on logout. Default sync interval is 5 minutes. `/etc` gets synced more often, more on that below
* Extend it through the `OVERLAY_DIRS` array

### TMPFS Mounts for Temporary Data

RAM storage for temporary files, cleaned automatically on logout:
* `/tmp` (5 GB) and `/var/tmp` (1 GB)
* `/var/cache` (2 GB) and `/home/$USER/.cache` (2 GB per user)

### Persistent Caches (Always on Disk)

Essential caches bind-mounted to `/persist` to survive reboots:
* System: `/var/cache/pacman`
* User: `paru`, `nvidia`, `mesa_shader_cache`, `mesa_shader_cache_db`
* Migrated automatically on first run

### Excluded from Overlay (Remain on Disk)

* `/home` - User data, too large and too important
* `/var/lib` - Databases, flatpak packages, too large
* `/opt`, `/usr/local` - Large applications
* `/proc`, `/sys`, `/dev`, `/run` - Virtual filesystems
* `/mnt`, `/media`, `/boot` - Mount points

Within an overlaid directory, `/var/log/journal` is excluded from both sync and the RAM pre-population step (`EXCLUDED_FROM_SYNC`). It's high-churn, journald manages its own persistence and there's no reason to copy it into RAM if it never gets written back to disk anyway.

`EXCLUDED_FROM_SYNC` also lists `/var/lib/systemd`, left over from when `/var/lib` used to be part of the overlay too. Since `/var/lib` isn't in `OVERLAY_DIRS` by default, that entry doesn't currently match anything. It only matters if you add `/var/lib` back yourself.

### Cleanup & Safety

* **Periodic cleanup:** Removes stale files older than 5 minutes, checked every 30 seconds
* **Periodic sync:** Syncs RAM changes back to disk every 5 minutes while a session is active (`SYNC_INTERVAL`, set to `0` to disable), plus a final sync at logout. This bounds how much a crash or power loss mid-session can cost you. `/etc` gets tighter coverage on top of that: a dedicated watcher (`EAGER_SYNC_DIRS`) checks every `CLEAN_INTERVAL` (30 seconds by default) and syncs pending `/etc` changes right away. The 5-minute interval is just the fallback cadence for `/var/log` and anything else added to the overlay
* **File protection:** Skips files currently in use, checked via `lsof`, `fuser`, or `/proc`
* **Graceful shutdown:** Catches SIGTERM, SIGINT and SIGHUP, plus unexpected exits and syncs before exiting
* **Logging:** All operations are tracked in `/var/log/ramoverlay.log` for the current session. The previous session's log gets archived to `/var/log/ramoverlay.last.log` and the main log is truncated at the end of each session, so only the current and immediately prior session stick around
* **Memory-pressure warning:** Logs a warning if available RAM drops below 10% (`MEM_WARN_PERCENT`). Sustained pressure risks the OOM killer targeting Xorg or the compositor

### Mimalloc Integration

Preloads [mimalloc](https://github.com/microsoft/mimalloc) for `rsync`, `find` and `inotifywait` when available, cutting down on memory fragmentation during sync operations and long-lived event watching.

## Requirements

* **Commands:** `rsync` is required in practice. The periodic and logout sync of `/etc` and `/var/log` has no fallback, so without it those changes never make it to disk and get lost when the session ends. The first-run migration of persistent caches into `/persist` falls back to `mv` if `rsync` is missing, but that's a separate code path and doesn't cover the overlay sync. `lsof` and `fuser` are optional too, falls back to `/proc` scanning if neither is present
* **RAM:** 16GB+ recommended (typical usage: 200MB-2GB)
* **Filesystem:** Supports OverlayFS (ext4, btrfs, xfs, f2fs)
* **Optional:** `libmimalloc.so` for faster syncs
* **Optional:** Kernel 6.3+ for the tmpfs `noswap` mount option. Tried first, falls back cleanly on older kernels, so it's not required

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

Registering and enabling it (compiling the service database, adding it to whatever bundle your other longruns are in) varies by which version of Artix's `s6-rc` frontend you're running. Use the same `s6 set` / `s6 live install` workflow you already use for other services, then confirm it's running with `s6-rc-db list` or whatever check you normally use.

**Note:** Runs as a daemon, automatically managing overlay lifecycle based on user sessions.

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
BIND_MOUNTED_USER_CACHE=(paru nvidia mesa_shader_cache mesa_shader_cache_db)
```

**Tmpfs sizes:**
```bash
OVERLAY_BASE_SIZE="50%"  # Ceiling for the RAM overlay itself, as % of total RAM
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
EAGER_SYNC_DIRS=("/etc") # These dirs are also synced within CLEAN_INTERVAL of a change
```

## Monitoring

The daemon has its own status commands, usually more useful than checking mounts by hand:

```bash
# Overlay/mount health, pending RAM-entry counts, watcher status
sudo ephemeral-overlay status

# Trigger a manual sync on demand
sudo ephemeral-overlay sync

# Show the last N lines of the log (default 50)
ephemeral-overlay log [N]
```

You can also inspect things directly:

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
* Allocated: 50% of total RAM by default (`OVERLAY_BASE_SIZE`). That's a ceiling, not a reservation. Tmpfs only uses what's actually written

---

## How It Works

1. **Wait:** Daemon waits for user login
2. **Activate:** On login, creates RAM overlay and tmpfs mounts
3. **Operate:** All writes to overlaid directories go to RAM (9.3x faster)
4. **Cleanup:** Daemon removes stale temp files every 30 seconds
5. **Sync:** Incrementally syncs changed files back to disk every 5 minutes while the session is active, with `/etc` synced eagerly within about 30 seconds of a change, then a final full sync on logout
6. **Sleep:** Once no users have been detected for 3 consecutive checks 10 seconds apart, unmounts everything, frees RAM and waits for the next login

## Performance (vs SATA SSD)

| Metric | SATA Disk | RAM Overlay | Improvement |
|--------|-----------|-------------|-------------|
| Random Write IOPS | 94,600 | 877,000 | **9.3x faster** |
| Write Latency | 3.74ms | 0.52ms | **7.2x lower** |
| Sequential Write | 530 MB/s | 2,128 MB/s | **4x faster** |
| RAM Overhead | 0 | ~200MB | Minimal |

*Benchmarked on Samsung 870 EVO SATA SSD.* The RAM vs SSD gap is a hardware property, not something any of the settings above change.

## Safety Features

* **Best-effort persistence:** Changes sync to disk periodically during the session and again on logout or shutdown. Sync errors are detected and logged, but there's no retry logic or verification pass afterward
* **File-in-use detection:** Never deletes files that are currently open
* **Rsync verification:** Checks rsync exit codes, captured independently of `tee` and logs any sync failures
* **Protected directories:** `/home` is never overlaid by default. It's just not in `OVERLAY_DIRS`. There's no code-level check blocking it either, so treat this as a strong default, not a hard guarantee. Don't add `/home` to `OVERLAY_DIRS` yourself
* **Fallback mechanisms:** Multiple methods for file-in-use detection

**Signal handling:** The daemon gracefully handles SIGTERM, SIGINT and SIGHUP and also traps `EXIT` directly, so even an unexpected termination triggers a sync attempt before the process dies, not just a clean signal. The main loop sleeps in 1-second increments instead of one long `sleep N` call. Bash only checks for a pending trap between commands, so a signal arriving mid-sleep would otherwise queue for up to `CLEAN_INTERVAL` (30 seconds by default) before shutdown even started. That's long enough for a process supervisor with a shorter kill-escalation timeout to SIGKILL the daemon first and skip the sync entirely.

**OverlayFS semantics:** Per-file writes are atomic through OverlayFS, but the sync itself isn't crash-safe. A power loss mid-rsync can leave the filesystem partially written. That's not a mount-level risk though. The overlay is mounted once at login and unmounted once at logout and every sync in between (periodic, eager, or final) is just an rsync from the RAM upper layer onto the on-disk copy, no unmount or remount involved.

The mount also deliberately skips the `volatile` option. It looked like a good fit at first. The upper layer is tmpfs and never durable across a reboot anyway, so OverlayFS's own sync/fsync bookkeeping on it seemed pointless. Testing showed otherwise: it writes a marker into workdir that makes the kernel refuse every later mount using that workdir, with or without `volatile`, until it's wiped. The RAM tmpfs and its workdir get destroyed on a clean logout and rebuilt from scratch at the next login. This only becomes a problem if the daemon is killed uncleanly and a restart inherits the still-mounted tmpfs.
