# nebula-sync

Keep two (or more) Pi-holes in sync — adlists, whitelists, and settings changed on the primary are copied to the replicas automatically. This is the actively maintained replacement for Gravity Sync (retired, Pi-hole v5 only) and Orbital Sync (archived), built for Pi-hole v6's API. Pairs well with [keepalived](/Pi-Guide/keepalived.md).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

- Two Pis running [Pi-hole](/Pi-Guide/Pi-hole.md) v6+
- [Docker](/Pi-Guide/Docker.md) on one of them (nebula-sync only runs in one place and talks to both Pi-holes over their APIs)

## Installation

1. Create a folder and a compose file:
   ```bash
   mkdir ~/nebula-sync
   nano ~/nebula-sync/docker-compose.yml
   ```
1. Paste the following in, replacing the IP addresses and Pi-hole web interface passwords. The format is `http://[ADDRESS]|[PASSWORD]`:
   ```yaml
   services:
     nebula-sync:
       image: ghcr.io/lovelaze/nebula-sync:latest
       container_name: nebula-sync
       restart: unless-stopped
       environment:
         - PRIMARY=http://[PRIMARYPIIPADDRESS]|[PASSWORD]
         - REPLICAS=http://[SECONDARYPIIPADDRESS]|[PASSWORD]
         - FULL_SYNC=true
         - RUN_GRAVITY=true
         - CRON=0 * * * *
   ```
   - `FULL_SYNC=true` syncs settings as well as adlists/domains; set it to `false` to sync only adlists, domains, groups, and clients.
   - `CRON=0 * * * *` syncs every hour — adjust to taste.
   - Multiple replicas are comma-separated in `REPLICAS`.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then start it:
   ```bash
   cd ~/nebula-sync
   docker compose up -d
   ```
1. If you'd rather not put your main web password in the file, create an app password in Pi-hole (Settings > Web Interface/API > Configure app password) and use that instead. When using app passwords, also enable API sudo mode on each replica:
   ```bash
   sudo pihole-FTL --config webserver.api.app_sudo true
   ```

## Testing

1. Check the logs for a successful sync:
   ```bash
   docker logs nebula-sync
   ```
1. Add an adlist or whitelist entry on the primary Pi-hole, then restart the container (or wait for the next scheduled sync) and confirm it appears on the secondary:
   ```bash
   docker restart nebula-sync
   ```

## Sources

- https://github.com/lovelaze/nebula-sync
- https://docs.pi-hole.net/api/
