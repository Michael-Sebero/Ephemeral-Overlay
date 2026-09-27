## Summary
Ephemeral Overlay moves writes to specified directories into RAM, reducing disk I/O and increasing system responsiveness. Changes are synced back to disk during the session and at logout.

## Features

### RAM Overlay for System Directories

* Critical system directories run from RAM through OverlayFS:
  * `/etc` - System configuration
  * `/var/log` - System logs
* **Performance:** 9.3x faster random-write IOPS (877K vs 94.6K), 7.2x lower write latency (0.52ms vs 3.74ms). Reads of files that haven't been written this session still come from disk or the page cache
* **Efficiency:** Only files written during the session live in RAM. The first write to an existing file copies it into RAM once, in full
* **Automatic:** Activates on login, or once a display-manager greeter has been idle for 5 seconds (`GREETER_SETTLE_SECONDS`) so the overlay is already up when you log in. Syncs during the session and at logout, more on that below
* Extend it through the `OVERLAY_DIRS` array. Entries must be absolute paths without a trailing slash, can't be nested inside each other and can't contain commas, colons, backslashes or whitespace. Anything else is skipped with an error in the log

### TMPFS Mounts for Temporary Data

RAM storage for temporary files, discarded when the session ends:
* `/tmp` (5 GB) and `/var/tmp` (1 GB)
* `/var/cache` (2 GB) and `/home/$USER/.cache` (2 GB per user)

A directory that's already a mountpoint, like `/tmp` from fstab, is left alone and never unmounted by the daemon. Two cases keep `/tmp` on disk for the current session:
* **An X server is already running** when the overlay starts (`SKIP_TMP_WITH_LIVE_X`). That's always the case when it starts behind a greeter. A fresh TMPFS over `/tmp` would hide the server's socket and lock file, which breaks clients that need the socket file (Flatpak's X11 passthrough) and lets a later X server pick a display number that's already taken. To keep `/tmp` in RAM from boot, mount it from fstab instead: `tmpfs /tmp tmpfs defaults,nosuid,nodev,size=5G,mode=1777 0 0`
* **A user session is already running** when the daemon starts (`LATE_START_SKIP_TMPFS`). `~/.cache` stays on disk then too, its persistent subdirectories are still bind-mounted

Directories listed in `KEEP_VAR_CACHE_DIRS` (default `lightdm`) are recreated on the `/var/cache` TMPFS with their original owner and mode.

### Persistent Caches (Always on Disk)

Essential caches bind-mounted from `/persist` to survive reboots:
* System: `/var/cache/pacman` and `/var/cache/xbps`, whichever exist on disk or are already in `/persist`
* User: `paru`, `nvidia`, `mesa_shader_cache`, `mesa_shader_cache_db`, `TauonMusicBox`
* Migrated automatically on first run with `mv`. With `/persist` on the same filesystem that's a rename, so even a multi-GB package cache moves instantly

### Excluded from Overlay (Remain on Disk)

* `/home` - User data, too large and too important
* `/var/lib` - Databases, flatpak packages, too large
* `/opt`, `/usr/local` - Large applications
* `/proc`, `/sys`, `/dev`, `/run` - Virtual filesystems
* `/mnt`, `/media`, `/boot` - Mount points

Within an overlaid directory, `/var/log/journal` is excluded from sync (`EXCLUDED_FROM_SYNC`). It's high-churn and journald manages its own persistence. Entries match whole path components, so `/var/log/journal` doesn't also catch something like `/var/log/journal-old`.

`EXCLUDED_FROM_SYNC` also lists `/var/lib/systemd`, left over from when `/var/lib` used to be part of the overlay too. Since `/var/lib` isn't in `OVERLAY_DIRS` by default, that entry doesn't currently match anything. It only matters if you add `/var/lib` back yourself.

**runit service directories:** When `runsvdir` is running and `/etc` is overlaid, `/etc/runit`, `/etc/sv` and `/etc/service` (`INIT_PROTECT_DIRS`) stay on disk. They're bound from disk back over the overlay and `runsvdir` is paused while `/etc` is swapped, so it never sees its service directories change underneath it and restart services. It resumes after 3 seconds even if the daemon dies mid-swap. Entries that are symlinks are skipped.

Anything in `EXCLUDED_FROM_OVERLAY` is handled the same way: bound from disk over the overlay, never in RAM and never synced.

### Cleanup & Safety

* **Periodic cleanup:** Every 30 seconds, deletes files older than 5 minutes (by modification time) that no process has open. Only TMPFS mounts get cleaned (`/tmp`, `/var/tmp`, `/var/cache`, `~/.cache`), whether the daemon mounted them or fstab did. A `/tmp` left on disk is never touched. Raise `STALE_MINUTES` if you build software in `/tmp`, since object files older than 5 minutes get deleted mid-build
* **Periodic sync:** Each overlaid directory has an inotify watcher that marks it dirty on any write, attribute change, create, delete or rename. `/etc` (`EAGER_SYNC_DIRS`) is checked every `CLEAN_INTERVAL` (30 seconds by default) and synced if dirty, everything else every `SYNC_INTERVAL` (5 minutes, `0` to disable). A directory nothing wrote to isn't synced at all. Without `inotifywait`, every check does a sync. A final sync runs at logout and shutdown. This bounds how much a crash or power loss mid-session can cost you
* **Deletions reach disk:** Files and directories deleted in RAM are removed from disk and a directory that was deleted and recreated doesn't keep its old files
* **File protection:** Cleanup skips files currently in use, checked via `lsof`, `fuser`, or `/proc`
* **Graceful shutdown:** Catches SIGTERM, SIGINT and SIGHUP, plus unexpected exits and syncs before exiting
* **Single instance:** A lock on `/var/run/ramoverlay/daemon.lock` makes a second copy exit right away
* **Logging:** All operations are tracked in `/var/log/ramoverlay.log` for the current session. The previous session's log gets archived to `/var/log/ramoverlay.last.log` and the main log is truncated at the end of each session, so only the current and immediately prior session stick around. The log lives inside the `/var/log` overlay, so the line or two written between the final sync of `/var/log` and its unmount are lost with the RAM layer
* **Memory-pressure warning:** Logs a warning once when available RAM drops below 10% (`MEM_WARN_PERCENT`) and another line when it recovers. Sustained pressure risks the OOM killer targeting Xorg or the compositor

### Mimalloc Integration

Preloads [mimalloc](https://github.com/microsoft/mimalloc) for `rsync`, `find` and `inotifywait` when available, cutting down on memory fragmentation during sync operations and long-lived event watching.

## Requirements

* **Commands:** `rsync` is required for the overlays. Without it the daemon only sets up the TMPFS mounts and logs that `/etc` and `/var/log` weren't overlaid, since their changes could never reach disk. `flock`, `findmnt` and `mountpoint` (util-linux) are required
* **RAM:** 16GB+ recommended
* **Filesystem:** Supports OverlayFS (ext4, btrfs, xfs, f2fs)
* **Optional:** `inotifywait` (inotify-tools) for dirty tracking. Without it, every sync interval does a full sync
* **Optional:** `lsof` or `fuser`. Falls back to `/proc` scanning if neither is present
* **Optional:** `getfattr`/`setfattr` (`attr`, `attr-progs` on Void), only used by `scrub` to clear leftover overlay attributes
* **Optional:** `libmimalloc.so` for faster syncs
* **Optional:** Kernel 6.4+ for the TMPFS `noswap` mount option. Tried first, falls back cleanly on older kernels, so it's not required

## Installation

```bash
sudo install -m 755 ephemeral-overlay /bin/ephemeral-overlay
```

`install` writes a new file instead of overwriting the old one in place. That matters when replacing a running copy: bash reads a script from disk as it executes, so overwriting it with `cp` can feed the running daemon lines from the new version.

Pick one of the start methods below, not two. A second copy exits on the lock and a supervisor restarts it every second, filling the log.

**On rc.local systems**, add to `/etc/rc.local`:

```bash
# Start ephemeral overlay daemon
if [ -x /bin/ephemeral-overlay ]; then
    ( sleep 20; setsid nohup /bin/ephemeral-overlay >/dev/null 2>&1 ) &
fi
```

The daemon writes `/var/log/ramoverlay.log` itself, so its own output goes to `/dev/null`. Redirecting it into the log file duplicated every line. That descriptor was also opened on disk before `/var/log` got overlaid, so the lines written through it were overwritten by the RAM copy at the next sync anyway.

**On runit (Void):**

```bash
sudo mkdir -p /etc/sv/ephemeral-overlay
printf '#!/bin/sh\nexec /bin/ephemeral-overlay >/dev/null\n' | sudo tee /etc/sv/ephemeral-overlay/run >/dev/null
sudo chmod 755 /etc/sv/ephemeral-overlay/run
sudo ln -s /etc/sv/ephemeral-overlay /var/service/
```

Void's shutdown stops services with `sv force-stop`, which waits 7 seconds (`SVWAIT`) and then kills whatever's still running. Give the final sync more time:

```bash
echo 'export SVWAIT=30' | sudo tee -a /etc/rc.conf
```

On Artix runit the same `run` file goes in `/etc/runit/sv/ephemeral-overlay`, enabled by linking that directory into `/etc/runit/runsvdir/default/`.

**On s6 (Artix)**, define it as a longrun:

```bash
sudo mkdir -p /etc/s6/sv/ephemeral-overlay
echo longrun | sudo tee /etc/s6/sv/ephemeral-overlay/type >/dev/null
printf '#!/bin/sh\nexec /bin/ephemeral-overlay >/dev/null\n' | sudo tee /etc/s6/sv/ephemeral-overlay/run >/dev/null
sudo chmod 755 /etc/s6/sv/ephemeral-overlay/run
```

Registering and enabling it (compiling the service database, adding it to whatever bundle your other longruns are in) varies by which version of Artix's `s6-rc` frontend you're running. Use the same `s6 set` / `s6 live install` workflow you already use for other services, then confirm it's running with `s6-rc-db list` or whatever check you normally use.

**Note:** Runs as a daemon, automatically managing overlay lifecycle based on user sessions.

## Upgrading From the Previous Version

1. Install the new file as shown above, then stop the old daemon. It syncs and unmounts on SIGTERM, even if it's stuck in the deadlock the old version could hit on its first sync. A supervisor restarts it from the new file.
   ```bash
   sudo kill -TERM "$(cat /var/run/ramoverlay-daemon.pid)"
   ```
2. Clean up what the old sync left on disk. It copied OverlayFS's deletion markers over the real files, so every file deleted during a past session now exists on disk as a `c 0,0` device node. They're hidden while the overlay is mounted but show up early in boot, before the first login. It also copied `trusted.overlay.*` attributes onto real directories. `scrub` removes both and prints a PASS summary:
   ```bash
   sudo ephemeral-overlay scrub
   ```

Runtime state now lives in `/var/run/ramoverlay/` (recorded mounts, locks, dirty flags, watcher PIDs). The state file and PID file keep their old paths.

## Configuration

Edit `/bin/ephemeral-overlay`:

**Directories to overlay:**
```bash
OVERLAY_DIRS=(
    "/etc"
    "/var/log"
    # "/opt"          # Add more as needed
)
EAGER_SYNC_DIRS=("/etc")          # Also synced within CLEAN_INTERVAL of a change
EXCLUDED_FROM_SYNC=(              # Overlaid, but changes are never written back
    "/var/lib/systemd"
    "/var/log/journal"
)
EXCLUDED_FROM_OVERLAY=()          # Bound from disk over the overlay, never in RAM
```

**Persistent system caches:**
```bash
BIND_MOUNTED_VAR_CACHE=(pacman xbps)
```

**Persistent user caches:**
```bash
BIND_MOUNTED_USER_CACHE=(paru nvidia mesa_shader_cache mesa_shader_cache_db TauonMusicBox)
```

**TMPFS sizes:**
```bash
OVERLAY_BASE_SIZE="50%"  # Ceiling for the RAM overlay itself, as % of total RAM
TMP_SIZE="5G"
VAR_TMP_SIZE="1G"
VAR_CACHE_SIZE="2G"
USER_CACHE_SIZE="2G"
```

**Cleanup, sync and session settings:**
```bash
CLEAN_INTERVAL=30          # Cleanup and /etc sync check, in seconds
STALE_MINUTES=5            # Delete unopened TMPFS files older than this
SYNC_INTERVAL=300          # Sync the other dirs every 5 min if dirty; 0 to disable
GREETER_SETTLE_SECONDS=5   # Start behind an idle greeter after this long; 0 waits for a real login
LOGOUT_CONFIRM_CHECKS=3    # Consecutive "no users" checks before teardown
LOGOUT_CONFIRM_INTERVAL=10 # Seconds between those checks
SKIP_TMP_WITH_LIVE_X=1     # Leave /tmp on disk while an X server is running
LATE_START_SKIP_TMPFS=1    # Leave /tmp and ~/.cache on disk if a session is already running
```

## Monitoring

The daemon has its own status commands, usually more useful than checking mounts by hand:

```bash
# PASS/FAIL per overlaid directory: overlay mounted, watcher, unsynced changes, entries in RAM
sudo ephemeral-overlay status

# Trigger a manual sync on demand, prints PASS or FAIL
sudo ephemeral-overlay sync

# Remove deletion markers and overlay attributes left on disk by older versions
sudo ephemeral-overlay scrub

# Show the last N lines of the log (default 50)
ephemeral-overlay log [N]
```

You can also inspect things directly:

```bash
# View active overlays
findmnt -t overlay

# Check RAM usage
df -h /ram_overlay

# See what's in RAM (/ram_overlay is root-only, so the glob has to expand as root)
sudo sh -c 'du -sh /ram_overlay/upper/*'

# Monitor activity
tail -f /var/log/ramoverlay.log
```

**RAM usage** is whatever got written during the session: the files changed under `/etc` and `/var/log`, plus everything in the TMPFS mounts. `OVERLAY_BASE_SIZE` (50% of total RAM by default) is a ceiling, not a reservation. TMPFS only uses what's actually written.

---

## How It Works

1. **Wait:** Daemon waits for a login (tty, elogind session, or an X or Wayland socket owned by a regular user) or an idle greeter
2. **Activate:** Creates the TMPFS mounts, then the RAM overlays. Every mount is recorded so teardown only touches what the daemon created
3. **Operate:** All writes to overlaid directories go to RAM (9.3x faster). Untouched files are still read from disk
4. **Cleanup:** Daemon removes stale temp files every 30 seconds
5. **Sync:** Writes dirty directories back to disk, `/etc` within about 30 seconds of a change and the rest every 5 minutes, then a final sync on logout
6. **Sleep:** Once no users are seen at a 30-second check and at two more checks 10 seconds apart, syncs, unmounts everything it mounted, frees RAM and waits for the next login

## Performance (vs SATA SSD)

| Metric | SATA Disk | RAM Overlay | Improvement |
|--------|-----------|-------------|-------------|
| Random Write IOPS | 94,600 | 877,000 | **9.3x faster** |
| Write Latency | 3.74ms | 0.52ms | **7.2x lower** |
| Sequential Write | 530 MB/s | 2,128 MB/s | **4x faster** |
| RAM Overhead | 0 | Files written this session | Minimal |

*Benchmarked on Samsung 870 EVO SATA SSD.* The RAM vs SSD gap is a hardware property, not something any of the settings above change. It applies to writes. Reads of files the session hasn't written go to the disk underneath, same as without the overlay.

## Safety Features

* **Best-effort persistence:** Changes sync to disk periodically during the session and again on logout or shutdown. Sync errors are detected and logged, but there's no retry logic or verification pass afterward
* **File-in-use detection:** Never deletes files that are currently open
* **Rsync verification:** Checks rsync's exit code. Code 24 (files that vanished mid-sync, normal for a live tree) counts as success, anything else logs rsync's error lines and marks the sync failed
* **Sync lock:** The daemon's syncs, `sync` and `scrub` share `/var/run/ramoverlay/sync.lock`, so two rsyncs never write the same disk tree at once
* **Mount tracking:** Every mount the daemon creates is recorded in `/var/run/ramoverlay/mounts` and torn down in reverse order. Mounts it didn't create are never unmounted. An overlay that's still busy at teardown is detached lazily and synced once more from RAM before its RAM is released. A restarted daemon takes over an overlay that's still mounted and syncs it right away instead of starting fresh
* **Protected directories:** `/home` is never overlaid by default. It's just not in `OVERLAY_DIRS`. There's no code-level check blocking it either, so treat this as a strong default, not a hard guarantee. Don't add `/home` to `OVERLAY_DIRS` yourself
* **Fallback mechanisms:** Multiple methods for file-in-use detection

**Deletion handling:** OverlayFS records a deleted file as a `0,0` character device (a whiteout) in the RAM layer and a replaced directory as an extended attribute. Copying those to disk turns deleted files into device nodes. Instead, for every directory that exists in RAM, the sync compares the disk copy with the live overlay view and removes whatever the live view no longer shows. Then it copies the RAM layer over, skipping device nodes and `trusted.overlay.*` attributes. Comparing against the live view also covers what whiteouts miss: a file deleted inside a directory that was created this session and already synced leaves no whiteout at all.

**Mount options:** The overlay is mounted with `redirect_dir=nofollow,metacopy=off,index=off,xino=auto`. With redirects, renaming a directory stores a pointer to its old location instead of moving it and a file-level sync can't follow that. With metacopy, the RAM copy of a file can hold metadata but no data, so rsync would write zeros over the real file. The fallback option sets exist for older kernels that don't know these options.

**Signal handling:** The daemon traps SIGTERM, SIGINT, SIGHUP and `EXIT`, so even an unexpected termination triggers a sync attempt before the process dies, not just a clean signal. It waits with `read -t` on an idle pipe instead of `sleep` and a trapped signal interrupts that wait immediately, so shutdown starts right away instead of after the current sleep. Once the final sync has started, further signals are ignored so a second SIGTERM can't cut it short. A supervisor that escalates to SIGKILL still can, which is what `SVWAIT` above is for.

**OverlayFS semantics:** The overlay is mounted once at login and unmounted once at logout. Every sync in between (periodic, eager or final) works on the RAM upper layer and the on-disk copy, no unmount or remount involved. rsync replaces each file atomically (temp file plus rename), but the sync as a whole isn't crash-safe. A power loss mid-sync can leave some files updated and others not, plus a stray `.name.XXXXXX` temp file. Processes that opened a file before the overlay existed (the greeter's `Xorg.0.log`, svlogd's `current`) keep writing straight to disk. The sync leaves those files alone unless the same file is also written through the overlay, in which case the RAM copy wins.

The mount also deliberately skips the `volatile` option. It looked like a good fit at first. The upper layer is TMPFS and never durable across a reboot anyway, so OverlayFS's own sync/fsync bookkeeping on it seemed pointless. Testing showed otherwise: it writes a marker into workdir that makes the kernel refuse every later mount using that workdir, with or without `volatile`, until it's wiped. The RAM TMPFS and its workdir get destroyed on logout and rebuilt from scratch at the next login. A restarted daemon takes over a still-mounted overlay instead of remounting it and refuses to reuse a TMPFS at `/ram_overlay` it didn't record mounting, so a stale workdir never gets remounted.

## Troubleshooting

* **`elogind is already running as PID` repeating in dmesg:** Not caused by this daemon. It only reads elogind's session files and never calls `loginctl`. On Void it comes from enabling the elogind runit service alongside sddm, whose run script starts elogind through D-Bus. The two race and runsv restarts the losing copy every second. Remove the service with `sudo rm /var/service/elogind` and reboot. On Artix, elogind is meant to run as a service, so leave it enabled there
* **`/tmp` isn't a TMPFS:** Check the log. If an X server was already running at start, `/tmp` is left on disk on purpose, see TMPFS Mounts above
* **`Another instance holds /var/run/ramoverlay/daemon.lock` repeating in the log:** Two start methods are active, see Installation
* **A sync reported FAIL:** `ephemeral-overlay log 100` shows the rsync error lines
