# Sonarr, Radarr & Prowlarr

Automated media library management. **Sonarr** handles TV shows, **Radarr** handles movies, and **Prowlarr** manages the indexers both of them search — configure an indexer once in Prowlarr and it syncs to the others automatically. They monitor your library, fetch what's missing, rename it into a consistent folder structure, and hand it off to [Jellyfin](/Pi-Guide/Jellyfin.md).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Folder Layout](#folder-layout)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. First Run](#1-first-run)
  - [2. Root Folders](#2-root-folders)
  - [3. Download Client](#3-download-client)
  - [4. Connect Prowlarr to Sonarr and Radarr](#4-connect-prowlarr-to-sonarr-and-radarr)
  - [5. Indexers](#5-indexers)
  - [6. Restrict Allowed Hostnames (Optional)](#6-restrict-allowed-hostnames-optional)
- [Seerr (Optional)](#seerr-optional)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [NAS](/Pi-Guide/NAS.md) — media is large and belongs on an external drive
- [Jellyfin](/Pi-Guide/Jellyfin.md) (recommended) — this guide uses the same media folders, so the two work together with no extra configuration
- A download client (see [Download Client](#3-download-client)) — the *arr apps find and organize media but don't download it themselves

A note on hardware: these are three .NET applications running alongside whatever else you host. A Pi 4 with 4GB handles all three comfortably; on a 1GB Pi, expect it to be tight and consider running them on a separate machine.

## Folder Layout

This is the one decision worth getting right up front, because changing it later means re-importing your whole library.

All three containers mount the **entire drive** at `/data` rather than mounting individual media folders. That keeps downloads and your library on the same filesystem inside the container, which lets the *arr apps use **hardlinks** and **atomic moves** — importing a finished download becomes instant and uses no extra disk space, instead of copying the file and doubling its size.

Assuming the drive is mounted at `/mnt/sda1` per the NAS guide:

```
/mnt/sda1/
├── downloads/          ← your download client writes here
└── media/
    ├── Movies/         ← Radarr's root folder
    └── Shows/          ← Sonarr's root folder
```

This deliberately keeps `media/Movies` and `media/Shows` exactly where the Jellyfin guide already expects them, so **no changes to your Jellyfin setup are needed** — it keeps reading `/mnt/sda1/media` as before.

Create the folders:

```bash
sudo mkdir -p /mnt/sda1/downloads /mnt/sda1/media/Movies /mnt/sda1/media/Shows
sudo chown -R $USER:$USER /mnt/sda1/downloads /mnt/sda1/media
```

## Installation

1. Create the config folders:
   ```bash
   mkdir -p ~/arr/sonarr ~/arr/radarr ~/arr/prowlarr
   ```
1. Find your user and group ID — you'll need them in the compose file so the containers write files your normal user can read:
   ```bash
   id
   ```
   Note the `uid=` and `gid=` values (usually both `1000`).
1. Create the compose file:
   ```bash
   nano ~/arr/docker-compose.yml
   ```
1. Paste the following in, replacing `PUID`/`PGID` with your values from above and `TZ` with your timezone:
   ```yaml
   services:
     prowlarr:
       image: lscr.io/linuxserver/prowlarr:latest
       container_name: prowlarr
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=America/Toronto
       volumes:
         - /home/pi/arr/prowlarr:/config
       ports:
         - 9696:9696
       restart: unless-stopped

     sonarr:
       image: lscr.io/linuxserver/sonarr:latest
       container_name: sonarr
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=America/Toronto
       volumes:
         - /home/pi/arr/sonarr:/config
         - /mnt/sda1:/data
       ports:
         - 8989:8989
       restart: unless-stopped

     radarr:
       image: lscr.io/linuxserver/radarr:latest
       container_name: radarr
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=America/Toronto
       volumes:
         - /home/pi/arr/radarr:/config
         - /mnt/sda1:/data
       ports:
         - 7878:7878
       restart: unless-stopped
   ```
   Defining all three in one file puts them on a shared Docker network, so they can reach each other by container name (`http://sonarr:8989`) — that matters in [step 4](#4-connect-prowlarr-to-sonarr-and-radarr).
1. Since the media lives on an external drive, follow [Docker Containers Depending on External Drive](/Pi-Guide/NAS.md#docker-containers-depending-on-external-drive) so Docker waits for the mount on boot. Without this, the containers start before the drive is ready and Sonarr/Radarr will see an empty library.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then start them:
   ```bash
   cd ~/arr
   docker compose up -d
   ```

The three web UIs are now at:

| Service | URL |
| --- | --- |
| Sonarr | `http://[PIIPADDRESS]:8989` |
| Radarr | `http://[PIIPADDRESS]:7878` |
| Prowlarr | `http://[PIIPADDRESS]:9696` |

> If a browser refuses to load these, check that it isn't forcing HTTPS before assuming the container is broken — these serve plain HTTP, and Safari in particular blocks that outright unless you turn off Settings → Apps → Safari → Privacy & Security → "Not Secure Connection Warning". Serving them over TLS with [NGINX](/Pi-Guide/NGINX.md) avoids the issue on every device.

## Configuration

### 1. First Run

Open each of the three UIs. Each will ask you to create a username and password on first launch — do this even on a trusted LAN, since these apps can write to your filesystem and trigger downloads.

### 2. Root Folders

Tell each app where its library lives. These are the paths *inside* the container, not on the Pi.

- **Sonarr** → Settings → Media Management → Root Folders → Add → `/data/media/Shows`
- **Radarr** → Settings → Media Management → Root Folders → Add → `/data/media/Movies`

While you're in Media Management, turn on **Rename Episodes** (Sonarr) / **Rename Movies** (Radarr). Consistent naming is what lets Jellyfin identify everything correctly.

### 3. Download Client

Sonarr and Radarr don't download anything themselves — they search indexers, then hand the result to a download client. You need one running before the stack does anything useful. The common choices are **qBittorrent** for torrents or **SABnzbd** for Usenet; both have LinuxServer.io images that drop into the same compose file, and both must write to `/data/downloads` so hardlinking works.

Once yours is running, add it in **both** Sonarr and Radarr under Settings → Download Clients → Add, using the container name as the host.

If you want the download client's traffic to go through a VPN, see [Gluetun](/Pi-Guide/Gluetun.md). It routes qBittorrent through the VPN and cuts it off entirely if the VPN drops, so nothing leaks out over your normal connection.

Prowlarr ships with no indexers and neither app comes with any content — you supply those, so make sure whatever you point it at is something you're entitled to access.

### 4. Connect Prowlarr to Sonarr and Radarr

This is what makes Prowlarr worth running: add an indexer once, and it pushes to both apps automatically.

1. In **Sonarr**, go to Settings → General and copy the **API Key**.
1. In **Prowlarr**, go to Settings → Apps → Add → Sonarr, and fill in:
   - Prowlarr Server: `http://prowlarr:9696`
   - Sonarr Server: `http://sonarr:8989`
   - API Key: the one you just copied
1. Click **Test** — it should go green — then Save.
1. Repeat for **Radarr**, using its API key from Radarr's Settings → General and `http://radarr:7878`.

Use the container names, not IP addresses. The containers resolve each other by name on the shared network, and this keeps working if the Pi's IP ever changes.

### 5. Indexers

In Prowlarr, go to Indexers → Add Indexer and add the ones you use. Each gets tested on save and then synced out to Sonarr and Radarr within a minute — you should see them appear under Settings → Indexers in both apps without adding them there manually.

### 6. Restrict Allowed Hostnames (Optional)

By default each app answers a request addressed to any hostname. That leaves an opening for "DNS rebinding", where a malicious web page open in a browser on your LAN tricks the browser into sending requests to the app. Each app can be told to accept only the names you actually use:

1. Stop the app (e.g. `docker compose stop radarr`) and open its config file (`~/arr/radarr/config.xml`).
1. Set the `<AllowedHosts>` line (add it inside `<Config>` if it's missing):
   ```xml
   <AllowedHosts>localhost,127.0.0.1,[PIIPADDRESS],[PIHOSTNAME],radarr</AllowedHosts>
   ```
   - **Include the container name** (`radarr`, `sonarr`, `prowlarr`). The apps reach each other by name. Leave it out and Prowlarr's sync and Seerr quietly stop working, with no error on the app's own page.
   - Add any domain you reach it through, e.g. behind [NGINX Proxy Manager](/Pi-Guide/NGINX.md#docker-alternative-nginx-proxy-manager).
1. Start it again, and repeat for the other two with their own container names.
1. Check that a known name is accepted and an unknown one refused:
   ```bash
   curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: radarr' http://localhost:7878/ping          # 200
   curl -s -o /dev/null -w '%{http_code}\n' -H 'Host: something-else' http://localhost:7878/ping # 400
   ```
   Then press **Test** on each app under Prowlarr's Settings → Apps.

## Seerr (Optional)

A friendly request front-end. Instead of you logging into Sonarr and Radarr to add things, household members browse a Netflix-style catalogue, click Request, and it lands in the right app automatically — with optional approval before anything downloads. It signs in against your Jellyfin users, so there are no separate accounts to hand out.

> **Naming:** this used to be **Overseerr** (Plex) and **Jellyseerr** (a fork adding Jellyfin/Emby support). In February 2026 the two projects merged into a single codebase called **Seerr**, which has all of Overseerr's features plus Jellyfin/Emby support. Both older names are deprecated — use Seerr for new installs. If you already run Jellyseerr or Overseerr, pointing the old config folder at the new image migrates it automatically on first start, but back up the config folder first.

1. Create the config folder. Seerr runs as the container's `node` user (UID 1000) rather than taking `PUID`/`PGID` like the LinuxServer images above, so ownership has to be set explicitly or it fails to write its database:
   ```bash
   mkdir -p ~/arr/seerr
   sudo chown -R 1000:1000 ~/arr/seerr
   ```
1. Add this service to `~/arr/docker-compose.yml` alongside the other three:
   ```yaml
     seerr:
       image: ghcr.io/seerr-team/seerr:latest
       container_name: seerr
       init: true
       environment:
         - TZ=America/Toronto
       volumes:
         - /home/pi/arr/seerr:/app/config
       ports:
         - 5055:5055
       restart: unless-stopped
   ```
   `init: true` is required — the image ships no init process of its own, and without this you accumulate zombie processes. Note there's no `/data` mount: Seerr only ever talks to the other apps over HTTP and never touches your media files.
1. Apply it:
   ```bash
   cd ~/arr
   docker compose up -d
   ```
1. Open `http://[PIIPADDRESS]:5055` and work through the setup wizard:
   - **Sign in with Jellyfin** — point it at `http://[PIIPADDRESS]:8096` and log in with your Jellyfin admin account.
     - If this Pi uses the [UFW firewall](/Pi-Guide/SSH-Hardening.md#firewall-ufw) and runs Jellyfin as in [its guide](/Pi-Guide/Jellyfin.md) (host networking), this step times out. Seerr's request comes from Docker's network, which your LAN rule doesn't cover. Allow it with `sudo ufw allow from 172.16.0.0/12 to any port 8096 proto tcp comment 'Seerr to Jellyfin'`.
   - **Services** — add Sonarr (`http://sonarr:8989`) and Radarr (`http://radarr:7878`) with the same API keys from [step 4](#4-connect-prowlarr-to-sonarr-and-radarr), and set the same root folders and quality profiles you configured there.
1. Invite household members under Users → Import from Jellyfin. Leave "Auto-Approve" off if you want requests to queue for your approval first.

## Testing

1. In **Radarr**, click Movies → Add New, search for something you own, pick a quality profile, and add it.
1. Go to Activity → Queue. If an indexer returns a result, it will hand off to your download client and appear here.
1. When it finishes, Radarr imports it into `/mnt/sda1/media/Movies` with a clean name.
1. In **Jellyfin**, scan the library — the new file shows up with correct metadata and artwork.
1. If you set up Seerr, request something from its UI instead and confirm it appears in Radarr's queue — that verifies the whole chain end to end.

To confirm hardlinks are working (the payoff from the `/data` mount), check that the imported file didn't consume a second copy:

```bash
du -sh /mnt/sda1/downloads /mnt/sda1/media
df -h /mnt/sda1
```

If the drive's used space didn't grow by the size of the file when it imported, hardlinking worked. If it doubled, something is mounted such that downloads and media are on different filesystems inside the container — re-check that both containers mount `/mnt/sda1:/data` and nothing narrower.

## Updating

```bash
cd ~/arr
docker compose pull && docker compose up -d
```

Or let [Watchtower](/Pi-Guide/Watchtower.md) do it automatically. Add all three to [Homepage](/Pi-Guide/Homepage.md) for one-click access, and to [Uptime Kuma](/Pi-Guide/Uptime-Kuma.md) if you want alerts when one falls over.

## Sources

- https://docs.linuxserver.io/images/docker-sonarr/
- https://docs.linuxserver.io/images/docker-radarr/
- https://docs.linuxserver.io/images/docker-prowlarr/
- https://wiki.servarr.com/
- https://trash-guides.info/Hardlinks/Hardlinks-and-Instant-Moves/
- https://docs.seerr.dev/getting-started/docker/
- https://docs.seerr.dev/migration-guide/
