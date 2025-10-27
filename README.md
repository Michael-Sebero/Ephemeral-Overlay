## Features

* **RAM overlay for the root filesystem:**

  * The root filesystem (`/`) is layered on top of a tmpfs-backed overlay stored in RAM
  * All filesystem changes (except excluded paths) occur in RAM and are **synced back to disk on shutdown**
  * The overlay is deactivated and cleaned automatically on shutdown
  * Excluded paths remain on disk to preserve user and system data:

    ```
    /home, /tmp, /var/tmp, /var/cache, /proc, /sys, /dev, /run, /mnt, /media, /boot
    ```

* **Tmpfs mounts for volatile directories:**

  * `/tmp` → 5 GB
  * `/var/tmp` → 1 GB
  * `/var/cache` → 2 GB
  * `/home/$USER/.cache` → 2 GB per user
  * These are recreated at startup and cleaned periodically

* **Persistent directories (bind-mounted on disk):**

  * `/var/cache/pacman`
  * `/home/$USER/.cache/paru`
  * `/home/$USER/.cache/nvidia`
  * `/home/$USER/.cache/mesa_shader_cache`
  * `/home/$USER/.cache/mesa_shader_cache_db`
  * Persistent directories are automatically migrated to `/persist` on first run and bound back to the live system

* **User `.cache` flexibility:**

  * Additional subdirectories can be **added or removed via the script** (edit the `BIND_MOUNTED_USER_CACHE` array)
  * Each user’s `.cache` directory is placed on tmpfs unless explicitly made persistent

* **Automatic user detection:**

  * Enumerates users with valid home directories (`UID ≥ 1000`) and sets up overlays and cache mounts automatically

* **File cleanup daemon:**

  * Runs periodically (default: every 60 s) and removes stale files older than 10 minutes
  * Skips files that are currently open using `lsof` (preferred), `fuser`, or manual `/proc` fd scanning

* **Safe open file detection:**

  * Uses `lsof` when available, falling back to `fuser` or `/proc` inspection to avoid deleting active files

* **Optional performance optimization:**

  * If `/usr/lib/libmimalloc.so` is present, the script preloads it for `rsync`, `find`, and `lsof` to improve performance

---

## Requirements

* `rsync`, `find`, and either `lsof` or `fuser`
* Optional: [mimalloc](https://github.com/microsoft/mimalloc) for improved sync and cleanup performance

---

## Installation

- Place command in `/bin`

```bash
sudo cp ephemeral-overlay /bin/
sudo chmod 755 /bin/ephemeral-overlay
```

- Add this section to `rc.local`

```
# EPHEMERAL OVERLAY
if [ -x /bin/ephemeral-overlay]; then
    ( sleep 20; setsid nohup /bin/ephemeral-overlay >> /var/log/ephemeral-overlay 2>&1 ) &
fi
```


