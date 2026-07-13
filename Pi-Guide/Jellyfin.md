# Jellyfin

Self-hosted media streaming — your movies, shows, and music from the Pi to any TV, phone, or browser. Free, no accounts or subscriptions (the open alternative to Plex).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Updating](#updating)
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

## Sources

- https://jellyfin.org/docs/general/installation/container/
- https://jellyfin.org/docs/general/server/media/movies/
