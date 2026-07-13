# Uptime Kuma

Self-hosted status dashboard that pings all your services and alerts you (email, Discord, Telegram, and ~100 others) when something goes down. A nice complement to [Grafana](/Pi-Guide/Grafana.md): Grafana shows hardware metrics, Uptime Kuma tells you the moment a service dies.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

```bash
docker run -d --name uptime-kuma --restart unless-stopped -p 3001:3001 -v uptime-kuma:/app/data louislam/uptime-kuma:1
```

## Configuration

1. Open `[PIIPADDRESS]:3001` in your address bar and create your admin account.
1. Click "Add New Monitor" for each service you run. Examples from this repo:

   | Service | Monitor Type | Target |
   | --- | --- | --- |
   | Pi-hole | HTTP(s) | `http://[PIIPADDRESS]:8080/admin` |
   | Portainer | HTTP(s) | `https://[PIIPADDRESS]:9443` (enable "Ignore TLS Error") |
   | Home Assistant | HTTP(s) | `http://[PIIPADDRESS]:8123` |
   | A second Pi | Ping | `[SECONDPIIPADDRESS]` |
   | DNS resolution | DNS | any hostname, resolver `[PIIPADDRESS]` |

1. Under Settings > Notifications, add how you want to be alerted, then attach the notification to your monitors.

Tip: if this Pi itself goes down, Uptime Kuma goes down with it — monitor the important Pi from a second device, or pair with [Watchdog](/Pi-Guide/Watchdog.md) so the Pi recovers on its own.

## Updating

```bash
docker stop uptime-kuma && docker rm uptime-kuma
docker pull louislam/uptime-kuma:1
docker run -d --name uptime-kuma --restart unless-stopped -p 3001:3001 -v uptime-kuma:/app/data louislam/uptime-kuma:1
```

Your monitors are kept in the `uptime-kuma` Docker volume.

## Sources

- https://github.com/louislam/uptime-kuma
