# Jellyfin

Self-hosted media streaming — your movies, shows, and music from the Pi to any TV, phone, or browser. Free, no accounts or subscriptions (the open alternative to Plex).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Updating](#updating)
- [Keeping the Database Small](#keeping-the-database-small)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [NAS](/Pi-Guide/NAS.md) (recommended) — media files are large, so keep them on an external drive; the NAS guide also makes it easy to drop new files in over the network

## Installation

1. Create a media folder on your external drive (this guide assumes it's mounted at `/mnt/sda1` per the NAS guide) and folders for Jellyfin's config:
   ```bash
   sudo mkdir -p /mnt/sda1/media
   mkdir -p ~/jellyfin/config ~/jellyfin/cache
   ```
1. Create the compose file:
   ```bash
   nano ~/jellyfin/docker-compose.yml
   ```
1. Paste the following in:
   ```yaml
   services:
     jellyfin:
       image: jellyfin/jellyfin
       container_name: jellyfin
       network_mode: host
       volumes:
         - /home/pi/jellyfin/config:/config
         - /home/pi/jellyfin/cache:/cache
         - /mnt/sda1/media:/media:ro
       restart: unless-stopped
   ```
1. Since the media lives on an external drive, follow [Docker Containers Depending on External Drive](/Pi-Guide/NAS.md#docker-containers-depending-on-external-drive) so Docker waits for the mount on boot.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then start it:
   ```bash
   cd ~/jellyfin
   docker compose up -d
   ```
1. If you use the [UFW firewall](/Pi-Guide/SSH-Hardening.md#firewall-ufw), allow Jellyfin's ports. `network_mode: host` means Jellyfin listens directly on the Pi, so unlike most containers it **is** blocked by UFW:
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 8096 proto tcp comment 'Jellyfin'
   sudo ufw allow from 192.168.50.0/24 to any port 7359 proto udp comment 'Jellyfin discovery'
   ```
   Port 7359 lets the TV and phone apps find the server on their own. Without it you have to type the address in.

## Configuration

1. Open `[PIIPADDRESS]:8096` in your address bar and go through the setup wizard.
1. Add a library and point it at `/media` (organize files as `media/Movies`, `media/Shows`, etc. — Jellyfin's naming docs under Sources explain the expected layout).

Note on performance: the Pi has no usable hardware video encoding for Jellyfin, so avoid transcoding — keep media in widely supported formats (H.264/AAC MP4/MKV) and clients will "direct play" them smoothly. If a video stutters, it's likely being transcoded on the CPU.

## Testing

Install the Jellyfin app on your phone/TV (or use the browser), enter `http://[PIIPADDRESS]:8096` as the server, and play something.

## Updating

```bash
cd ~/jellyfin
docker compose pull && docker compose up -d
```

If [Watchtower](/Pi-Guide/Watchtower.md) updates Jellyfin automatically, be aware that plugins are built against a specific Jellyfin version. A major update can leave a plugin disabled or broken until the plugin itself is updated. If a plugin's feature vanishes overnight, check Dashboard → Plugins before anything else. If you depend on a plugin, you may prefer to update Jellyfin by hand.

## Keeping the Database Small

Jellyfin's database (`~/jellyfin/config/data/jellyfin.db`) grows over time and never shrinks by itself. Deleted rows leave free space behind that SQLite keeps rather than returning to the disk. Live TV is the worst case, since the program guide is rewritten constantly: on one server with a large channel lineup the database grew from 120 MB to 2.2 GB in under a week, and about 40% of that was free space.

To check how much space a cleanup would win back, and to reclaim it:

1. **Stop Jellyfin first.** The database uses WAL mode, so the files on disk aren't complete while it's running, and editing them underneath it can corrupt them:
   ```bash
   cd ~/jellyfin && docker compose stop
   ```
1. Take a copy, then check the free space:
   ```bash
   sudo apt install sqlite3
   sudo cp -p ~/jellyfin/config/data/jellyfin.db ~/jellyfin.db.bak
   sudo sqlite3 ~/jellyfin/config/data/jellyfin.db 'PRAGMA page_count; PRAGMA freelist_count;'
   ```
   The second number divided by the first is the fraction a vacuum would reclaim. Under ~10% isn't worth the downtime. (`sudo` is needed because the container runs as root in this guide and owns the file. `-p` keeps that ownership on the copy.)
1. Vacuum and check it:
   ```bash
   sudo sqlite3 ~/jellyfin/config/data/jellyfin.db 'VACUUM; PRAGMA integrity_check;'
   ```
   It should print `ok`. If it prints anything else, put the copy back before starting Jellyfin: `sudo cp -p ~/jellyfin.db.bak ~/jellyfin/config/data/jellyfin.db`. Vacuuming needs free disk space about the size of the database while it runs.
1. Start Jellyfin again (`docker compose start`), check everything plays, then delete the backup.

If you ever edit the database by hand with `sqlite3`, run `PRAGMA foreign_keys=ON;` first. The `sqlite3` tool turns foreign keys off by default, so deleting a row won't remove the rows that depend on it.

## Sources

- https://jellyfin.org/docs/general/installation/container/
- https://jellyfin.org/docs/general/server/media/movies/
