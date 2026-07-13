# Homepage

A single dashboard page linking every service on your Pi — once you've followed a few guides here, this replaces remembering half a dozen `IP:port` combinations.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

1. Create the config folder:
   ```bash
   mkdir -p ~/homepage/config
   ```
1. Run Homepage. It's mapped to port `3100` here since [Grafana](/Pi-Guide/Grafana.md) uses 3000. The `HOMEPAGE_ALLOWED_HOSTS` variable is required — set it to exactly the address you'll type in the browser:
   ```bash
   docker run -d --name homepage --restart unless-stopped -p 3100:3000 -e HOMEPAGE_ALLOWED_HOSTS=[PIIPADDRESS]:3100 -v /home/pi/homepage/config:/app/config -v /var/run/docker.sock:/var/run/docker.sock:ro ghcr.io/gethomepage/homepage:latest
   ```

## Configuration

Everything is configured with YAML files in `~/homepage/config` (created on first run). Edit the services list:

```bash
nano ~/homepage/config/services.yaml
```

Example matching this repo's guides:

```yaml
- Network:
    - Pi-hole:
        href: http://[PIIPADDRESS]:8080/admin
        description: Ad-blocking
    - Portainer:
        href: https://[PIIPADDRESS]:9443
        description: Docker management

- Monitoring:
    - Grafana:
        href: http://[PIIPADDRESS]:3000
        description: Hardware metrics
    - Uptime Kuma:
        href: http://[PIIPADDRESS]:3001
        description: Service status

- Home:
    - Home Assistant:
        href: http://[PIIPADDRESS]:8123
        description: Smart home
```

Changes appear on refresh — no restart needed. Homepage also has service widgets (live stats from Pi-hole, Portainer, immich, etc.) and Docker auto-discovery via the mounted socket; see the docs under Sources.

## Testing

Open `[PIIPADDRESS]:3100` in your address bar. If you get a host-validation error instead of the dashboard, the address doesn't match `HOMEPAGE_ALLOWED_HOSTS` — recreate the container with the exact host:port you're using.

## Sources

- https://gethomepage.dev/installation/docker/
- https://gethomepage.dev/configs/services/
