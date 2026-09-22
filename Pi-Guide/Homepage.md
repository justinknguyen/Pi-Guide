# Homepage

A single dashboard page linking every service on your Pi — once you've followed a few guides here, this replaces remembering half a dozen `IP:port` combinations.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Safer Docker Access](#safer-docker-access)
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

If you use the [UFW firewall](/Pi-Guide/SSH-Hardening.md#firewall-ufw), Homepage's widgets and status checks that call `[PIIPADDRESS]:[PORT]` are blocked. Homepage runs in Docker, so those calls come from Docker's network, not your LAN. Links you click still work, since those go from your browser. Allow each port a widget uses:

```bash
sudo ufw allow from 172.16.0.0/12 to any port [PORT] proto tcp comment 'Homepage widgets'
```

If `docker network inspect` shows the container on a network outside `172.16.0.0/12`, add the same rule for that subnet — see [the note in SSH Hardening](/Pi-Guide/SSH-Hardening.md#firewall-ufw).

## Safer Docker Access

The install command above mounts Docker's socket so Homepage can show container status. Anything that can talk to that socket can create a privileged container and take over the Pi, and the `:ro` doesn't prevent that: it stops the socket *file* being replaced, not API calls being sent through it. On a dashboard only you can reach, that's an acceptable trade. If you ever put Homepage behind [NGINX Proxy Manager](/Pi-Guide/NGINX.md#docker-alternative-nginx-proxy-manager) or anywhere else outside your LAN, a single bug in Homepage becomes a takeover of the Pi.

A socket proxy sits in between and only passes the read-only calls Homepage needs:

1. Create a network for the two containers, and run the proxy on it. It isn't published to the Pi, so nothing outside that network can reach it:
   ```bash
   docker network create homepage
   docker run -d --name dockerproxy --restart unless-stopped --network homepage -e CONTAINERS=1 -e POST=0 -v /var/run/docker.sock:/var/run/docker.sock:ro ghcr.io/tecnativa/docker-socket-proxy:latest
   ```
   `CONTAINERS=1` allows listing containers; `POST=0` blocks everything that would change anything.
1. Recreate Homepage on the same network, **without** the socket:
   ```bash
   docker stop homepage && docker rm homepage
   docker run -d --name homepage --restart unless-stopped --network homepage -p 3100:3000 -e HOMEPAGE_ALLOWED_HOSTS=[PIIPADDRESS]:3100 -v /home/pi/homepage/config:/app/config ghcr.io/gethomepage/homepage:latest
   ```
1. Point Homepage at the proxy in `~/homepage/config/docker.yaml`:
   ```yaml
   my-docker:
     host: dockerproxy
     port: 2375
   ```
1. Check it's doing its job. Listing works, changing things is refused:
   ```bash
   docker exec homepage wget -qO- http://dockerproxy:2375/containers/json | head -c 100; echo
   docker exec homepage wget -qO- --post-data= http://dockerproxy:2375/containers/create; echo "exit $?"
   ```
   The first prints the start of a JSON list; the second fails with `403 Forbidden`.

The same idea applies to any dashboard that asks for the Docker socket.

## Testing

Open `[PIIPADDRESS]:3100` in your address bar. If you get a host-validation error instead of the dashboard, the address doesn't match `HOMEPAGE_ALLOWED_HOSTS` — recreate the container with the exact host:port you're using.

## Sources

- https://gethomepage.dev/installation/docker/
- https://gethomepage.dev/configs/services/
- https://gethomepage.dev/configs/docker/
- https://github.com/Tecnativa/docker-socket-proxy
