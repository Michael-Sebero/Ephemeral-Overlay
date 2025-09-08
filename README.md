
## How Tmpfs Overlay Works
Temporary directories (`/tmp`, `/var/tmp`, `/var/log`, `/var/cache`, `/home/$USER/.cache/`) are mounted as tmpfs to leverage RAM for high-speed file storage. Each mount has a predefined limit (`/tmp` = 5G, `/var/tmp` = 1G, `/var/log` = 512M, `/var/cache` = 2G, `/home/$USER/.cache` = 2G). Essential directories `/var/cache/pacman`, `/home/$USER/.cache/paru`, `/home/$USER/.cache/nvidia`, `/home/$USER/.cache/mesa_shader_cache`, `/home/$USER/.cache/mesa_shader_cache_db` are excluded and bind-mounted on local storage.

## Requirements
. mimalloc (optional)

## How to Install
. Place `tmpfs-overlay` in `/bin`

. Make the program executable

```
sudo chmod 755 /bin/tmpfs-overlay
```

. Add this section to `rc.local`

```
# TMPFS Overlay
if [ -x /bin/tmpfs-overlay ]; then
    ( sleep 0.1; setsid nohup /bin/tmpfs-overlay >/dev/null 2>&1 ) &
fi
```

. Restart system
