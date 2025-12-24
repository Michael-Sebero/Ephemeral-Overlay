## Features

### RAM Overlay for System Directories

* Critical system directories are overlaid in RAM using OverlayFS for maximum performance:
  * `/etc` - System configuration
  * `/var/log` - System logs
* **Performance:** 9.3x faster IOPS (877K vs 94.6K), 7.2x lower latency (0.52ms vs 3.74ms)
* **Efficiency:** Uses only ~200MB RAM for typical workloads (~1-2% of 16GB)
* **Automatic:** Activates on login, syncs on logout
* Easily extensible via the `OVERLAY_DIRS` array

### Tmpfs Mounts for Temporary Data

High-speed RAM storage for temporary files (automatically cleaned on logout):
* `/tmp` (5 GB) and `/var/tmp` (1 GB)
* `/var/cache` (2 GB) and `/home/$USER/.cache` (2 GB per user)

### Persistent Caches (Always on Disk)

Essential caches bind-mounted to `/persist` to survive reboots:
* System: `/var/cache/pacman`
* User: `paru`, `nvidia`, `mesa_shader_cache`, `TauonMusicBox`
* Automatically migrated on first run

### Excluded from Overlay (Remain on Disk)

* `/home` - User data (too large/important)
* `/var/lib` - Databases, flatpak packages (too large)
* `/opt`, `/usr/local` - Large applications
* `/proc`, `/sys`, `/dev`, `/run` - Virtual filesystems
* `/mnt`, `/media`, `/boot` - Mount points

### Intelligent Cleanup & Safety

* **Periodic cleanup:** Removes stale files >10 minutes old every 60 seconds
* **File protection:** Skips in-use files (via `lsof`, `fuser`, or `/proc`)
* **Graceful shutdown:** Catches SIGTERM/SIGINT/SIGHUP, syncs before exit
* **Comprehensive logging:** All operations tracked in `/var/log/ramoverlay.log`

### Optional Performance Boost

Preloads [mimalloc](https://github.com/microsoft/mimalloc) for `rsync` and `find` if available, reducing memory fragmentation during sync operations.

---

## Requirements

* **Commands:** `rsync`, `find`, and either `lsof` or `fuser`
* **RAM:** 16GB+ recommended (typical usage: 200MB-2GB)
* **Filesystem:** Supports OverlayFS (ext4, btrfs, xfs, f2fs)
* **Optional:** `libmimalloc.so` for faster syncs

---

## Installation

```bash
sudo cp ephemeral-overlay /bin/
sudo chmod 755 /bin/ephemeral-overlay
```

Add to `/etc/rc.local`:

```bash
# Start ephemeral overlay daemon
if [ -x /bin/ephemeral-overlay ]; then
    ( sleep 20; setsid nohup /bin/ephemeral-overlay >> /var/log/ramoverlay.log 2>&1 ) &
fi
```

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
BIND_MOUNTED_USER_CACHE=(paru nvidia mesa_shader_cache TauonMusicBox)
```

**Tmpfs sizes:**
```bash
TMP_SIZE="5G"
VAR_TMP_SIZE="1G"
VAR_CACHE_SIZE="2G"
USER_CACHE_SIZE="2G"
```

**Cleanup settings:**
```bash
CLEAN_INTERVAL=60        # Check every 60 seconds
STALE_MINUTES=10         # Delete files older than 10 minutes
```

---

## Monitoring

```bash
# View active overlays
mount | grep overlay

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
4. **Cleanup:** Daemon removes stale temp files every 60 seconds
5. **Sync:** On logout, syncs all RAM changes back to disk
6. **Sleep:** Unmounts everything, frees RAM, waits for next login

---

## Performance (vs SATA SSD)

| Metric | SATA Disk | RAM Overlay | Improvement |
|--------|-----------|-------------|-------------|
| Random Write IOPS | 94,600 | 877,000 | **9.3x faster** |
| Write Latency | 3.74ms | 0.52ms | **7.2x lower** |
| Sequential Write | 530 MB/s | 2,128 MB/s | **4x faster** |
| RAM Overhead | 0 | ~200MB | Minimal |

*Benchmarked on Samsung 870 EVO SATA SSD*

---

## Safety Features

* **No data loss:** All changes synced to disk on logout and shutdown
* **Signal handling:** Gracefully handles SIGTERM, SIGINT, and SIGHUP
* **File-in-use detection:** Never deletes files that are currently open
* **Rsync verification:** Checks exit codes and logs sync failures
* **Protected directories:** User data in `/home` is never overlaid
* **Atomic operations:** Uses proper OverlayFS semantics
* **Fallback mechanisms:** Multiple methods for file-in-use detection
