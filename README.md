## Features

- **RAM overlay of root filesystem:**  
  - The root filesystem (`/`) is overlaid in RAM using an overlay filesystem.  
  - Changes made to files in the overlay are stored in RAM and synced back to disk on logout.  
  - Excluded directories remain on disk: `/home`, `/tmp`, `/var/tmp`, `/var/cache`, `/proc`, `/sys`, `/dev`, `/run`, `/mnt`, `/media`, `/boot`.

- **Tmpfs mounts for key directories:**
  - `/tmp` (5 GB)  
  - `/var/tmp` (1 GB)  
  - `/var/cache` (2 GB)  
  - `/home/$USER/.cache` (2 GB per user)

- **Persistent subdirectories (bind-mounted to disk):**
  - `/var/cache/pacman`  
  - `/home/$USER/.cache/paru`  
  - `/home/$USER/.cache/nvidia`  
  - `/home/$USER/.cache/mesa_shader_cache`  
  - `/home/$USER/.cache/mesa_shader_cache_db`  

- **Excluded directories** (not included in overlay):  
  `/home`, `/tmp`, `/var/tmp`, `/var/cache`, `/proc`, `/sys`, `/dev`, `/run`, `/mnt`, `/media`, `/boot`

- **Automatic user detection:** Overlay activates on login and syncs + cleans RAM on logout.

- **Stale file cleanup:** Automatically removes files older than 10 minutes in tmpfs and user caches, skipping in-use files.

- **Open file detection:** Uses `lsof` (preferred) or `fuser` to safely skip active files.

- **Optional performance enhancement:** Preload `libmimalloc.so` for faster `rsync`, `find`, and `lsof` operations.

---

## Requirements

- Optional: [mimalloc](https://github.com/microsoft/mimalloc) for improved performance  

---

## Installation

1. Copy the script to `/bin` and make it executable:

```bash
sudo cp tmpfs-overlay /bin/
sudo chmod 755 /bin/tmpfs-overlay
