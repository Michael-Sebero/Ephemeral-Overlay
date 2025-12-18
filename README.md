# Ephemeral Overlay

A high-performance RAM overlay system for Linux that dramatically speeds up system operations by intercepting writes to frequently-modified directories and storing them in RAM.

## Features

### RAM Overlay for System Directories

* Frequently-modified system directories are overlaid in RAM using overlay filesystem for maximum performance:
  * `/etc` - System configuration files
  * `/var/log` - System logs
* All changes occur in RAM and are **automatically synced back to disk on user logout**
* **Performance gains:** 9.3x faster IOPS (877K vs 94.6K), 7.2x lower latency (0.52ms vs 3.74ms)
* **Efficiency:** Uses only ~200MB RAM for typical workloads (~1-2% of available RAM)
* The overlay system is automatically activated on user login and deactivated on logout
* Additional directories can be overlaid by editing the `OVERLAY_DIRS` array in the script

### Tmpfs Mounts for Volatile Directories

* `/tmp` - 5 GB
* `/var/tmp` - 1 GB
* `/var/cache` - 2 GB
* `/home/$USER/.cache` - 2 GB per user
* These provide instant access and are automatically cleaned on logout

### Persistent Cache Directories (Bind-Mounted to Disk)

* `/var/cache/pacman` - Package manager cache
* `/home/$USER/.cache/paru` - AUR helper cache
* `/home/$USER/.cache/nvidia` - NVIDIA shader cache
* `/home/$USER/.cache/mesa_shader_cache` - Mesa shader cache
* `/home/$USER/.cache/mesa_shader_cache_db` - Mesa shader database
* `/home/$USER/.cache/TauonMusicBox` - Music player cache
* Persistent directories are automatically migrated to `/persist` on first run and bind-mounted to their original locations

### Flexible Cache Configuration

* Additional cache subdirectories can be added or removed by editing:
  * `BIND_MOUNTED_VAR_CACHE` array for system-wide caches
  * `BIND_MOUNTED_USER_CACHE` array for per-user caches
* Each user's `.cache` directory is mounted on tmpfs unless explicitly made persistent

### Excluded Directories (Remain on Disk)

* `/home` - User data (too important and large for RAM overlay)
* `/tmp`, `/var/tmp`, `/var/cache` - Already managed by tmpfs
* `/proc`, `/sys`, `/dev`, `/run` - Virtual/runtime filesystems
* `/mnt`, `/media`, `/boot` - Mount points and boot files
* These directories are never overlaid to ensure data safety and system stability

### Automatic User Detection and Session Management

* Detects user login via `who` command and X11 session detection
* Automatically activates overlays when users log in
* Monitors for user logout and triggers sync/cleanup when all users log out
* Enumerates users with valid home directories (UID ≥ 1000) and sets up cache mounts automatically

### Intelligent File Cleanup Daemon

* Runs periodically (default: every 60 seconds) during user sessions
* Removes stale files older than 10 minutes from temporary directories
* Skips files currently in use via `lsof` (preferred), `fuser`, or `/proc` fd scanning
* Protects persistent bind-mounted directories from cleanup

### Safe Shutdown Handling

* Catches SIGTERM, SIGINT, and SIGHUP signals for graceful shutdown
* Automatically syncs all changes from RAM to disk before system shutdown/reboot
* Verifies rsync exit codes and logs any sync failures
* Ensures no data loss during unexpected shutdowns

### Optional Performance Optimization

* If `/usr/lib/libmimalloc.so` is present, the script preloads it for `rsync`, `find`, and `lsof` to improve performance
* Reduces memory fragmentation and improves sync speed during high-load operations

### Comprehensive Logging

* All operations logged to `/var/log/ramoverlay.log` with timestamps
* Tracks overlay activation, sync operations, cleanup events, and errors
* Useful for debugging and monitoring system behavior

---

## Requirements

* `rsync`, `find`, and either `lsof` or `fuser`
* Sufficient RAM for overlay operations (recommended: 16GB+, typical usage: 200MB-2GB)
* Root filesystem that supports overlay filesystem (ext4, f2fs, btrfs, xfs)
* Optional: [mimalloc](https://github.com/microsoft/mimalloc) for improved sync and cleanup performance

---

## Installation

Place the script in `/bin` or `/usr/local/bin`:

```bash
sudo cp ephemeral-overlay /bin/
sudo chmod 755 /bin/ephemeral-overlay
```

Add this section to `rc.local`:

```bash
# EPHEMERAL OVERLAY
if [ -x /bin/ephemeral-overlay ]; then
    ( sleep 20; setsid nohup /bin/ephemeral-overlay >> /var/log/ramoverlay.log 2>&1 ) &
fi
```

**Note:** The script runs as a daemon and automatically manages the overlay lifecycle based on user login/logout events.

---

## Configuration

Edit `/bin/ephemeral-overlay` to customize behavior:

### Overlay Directories
Which directories are overlaid in RAM:

```bash
OVERLAY_DIRS=(
    "/etc"
    "/var/log"
    # Add more directories as needed
    # "/opt"          # Uncomment to overlay /opt
    # "/usr/local"    # Uncomment to overlay /usr/local
)
```

### Persistent Caches (System-Wide)

```bash
BIND_MOUNTED_VAR_CACHE=(
    pacman
    # Add other /var/cache subdirectories
)
```

### Persistent Caches (Per-User)

```bash
BIND_MOUNTED_USER_CACHE=(
    paru
    nvidia
    mesa_shader_cache
    mesa_shader_cache_db
    TauonMusicBox
    # Add other ~/.cache subdirectories
)
```

### Tmpfs Sizes

```bash
TMP_SIZE="5G"
VAR_TMP_SIZE="1G"
VAR_CACHE_SIZE="2G"
USER_CACHE_SIZE="2G"
```

### Cleanup Settings

```bash
CLEAN_INTERVAL=60        # Cleanup runs every 60 seconds
STALE_MINUTES=10         # Files older than 10 minutes are removed
```

---

## Monitoring

### Check Overlay Status

```bash
# View active overlays
mount | grep overlay

# Check RAM usage
df -h /ram_overlay

# View what's stored in RAM
sudo du -sh /ram_overlay/upper/*

# Monitor the log
tail -f /var/log/ramoverlay.log
```

### Expected RAM Usage

* Typical workload: 200MB - 500MB (~1-2% of 16GB RAM)
* Heavy usage: 1GB - 2GB (~5-10% of 16GB RAM)
* The script allocates 90% of total RAM but only uses what's needed

---

## How It Works

1. **Startup:** Script waits for user login
2. **Activation:** When user logs in:
   * Creates tmpfs mount at `/ram_overlay`
   * Mounts overlay filesystem on `/etc` and `/var/log`
   * Sets up tmpfs for temporary directories
   * Configures persistent bind mounts
3. **During session:** 
   * All writes to `/etc` and `/var/log` go to RAM (9.3x faster)
   * Cleanup daemon runs every 60 seconds
   * System operates normally with enhanced performance
4. **Logout:** When all users log out:
   * Syncs all changes from RAM back to disk
   * Unmounts overlays
   * Frees RAM
   * Waits for next login

---

## Performance Benefits

Based on benchmarks with SATA SSD (Samsung 870 EVO):

| Metric | Disk (SATA) | RAM Overlay | Improvement |
|--------|-------------|-------------|-------------|
| Random Write IOPS | 94,600 | 877,000 | **9.3x faster** |
| Write Latency | 3.74ms | 0.52ms | **7.2x lower** |
| Sequential Write | 530 MB/s | 2,128 MB/s | **4x faster** |
| RAM Usage | 0 | ~200MB | Minimal overhead |

---

## Safety Features

* **No data loss:** All changes are synced to disk on logout and shutdown
* **Signal handling:** Gracefully handles SIGTERM, SIGINT, and SIGHUP
* **File-in-use detection:** Never deletes files that are currently open
* **Rsync verification:** Checks exit codes and logs sync failures
* **Protected directories:** User data in `/home` is never overlaid
* **Atomic operations:** Uses proper overlay filesystem semantics
* **Fallback mechanisms:** Multiple methods for file-in-use detection
