# Portainer

Provides a GUI to easily manage Docker containers.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

Install and run Portainer:

```bash
docker pull portainer/portainer-ce:latest
docker run -d -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
```

- Only port `9443` (the HTTPS web UI) is needed. Many guides also publish `8000`, but that's only for Portainer's Edge Agent, used to manage Docker on *remote* machines — leave it off unless you need that.
- Remember Docker's published ports [bypass UFW](/Pi-Guide/SSH-Hardening.md#firewall-ufw), and Portainer has full control of every container on the Pi. Don't forward this port on your router; reach it from outside with [Tailscale](/Pi-Guide/Tailscale.md) instead.

## Testing

You can now access the WebUI by typing `https://[PIIPADDRESS]:9443` into your address bar. Portainer uses its own self-signed certificate, so your browser will show a security warning the first time — this is expected; choose to continue anyway. Follow the link under Sources to learn how to use Portainer.

Create your admin account within a few minutes of starting the container. For security, Portainer stops accepting a first-time setup after about 5 minutes — if you see a timeout message, just `docker restart portainer` and try again.

## Updating

1. Stop and remove Portainer
   ```bash
   docker stop portainer
   docker rm portainer
   ```
1. Repeat the commands under [Installation](#installation).

Your settings and account live in the `portainer_data` volume, so removing and recreating the container is safe.

## Sources

- https://pimylifeup.com/raspberry-pi-portainer/
