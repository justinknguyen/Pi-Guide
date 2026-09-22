# Gluetun

Routes a download client like qBittorrent through a VPN. Gluetun is a small container that connects to your VPN provider; other containers join its network and can then reach the internet **only** through the VPN. If the VPN drops, they lose their connection entirely instead of carrying on over your normal one and exposing your home IP.

## Table of Contents

- [Prerequisites](#prerequisites)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Connect Sonarr and Radarr](#connect-sonarr-and-radarr)
- [Testing](#testing)
- [Rules for Containers Sharing Gluetun's Network](#rules-for-containers-sharing-gluetuns-network)
- [Why WireGuard, Not OpenVPN](#why-wireguard-not-openvpn)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [Sonarr, Radarr & Prowlarr](/Pi-Guide/Arr-Stack.md) — this guide adds to its compose file
- A VPN subscription from a provider Gluetun supports (NordVPN, Mullvad, ProtonVPN, Surfshark, PIA and many more — see the provider list under Sources), with **WireGuard** support

## How It Works

Normally each container gets its own network connection. With `network_mode: service:gluetun`, qBittorrent instead shares Gluetun's, the same way two programs on one computer share its network. Gluetun sets up a firewall inside that shared network that only lets traffic out through the VPN tunnel. So whether or not the VPN is up, qBittorrent has no other way out.

Two consequences shape everything below:

- qBittorrent's ports are published on the **gluetun** container, since that's whose network they're on.
- The two containers are tied together. Restarting or recreating Gluetun affects qBittorrent too — see [Rules for Containers Sharing Gluetun's Network](#rules-for-containers-sharing-gluetuns-network).

## Installation

1. Get your provider's WireGuard details. Each provider does this differently: some give you a WireGuard config file to download, others (NordVPN) need an API call to get your private key. Your provider's page on the Gluetun wiki (under Sources) says exactly which values it needs. Usually that's a private key, and sometimes an address like `10.5.0.2/32`.
1. Put the key in a `.env` file next to the compose file, not in the compose file itself — see [Keep Secrets in a `.env` File](/Pi-Guide/Docker.md#keep-secrets-in-a-env-file):
   ```bash
   nano ~/arr/.env
   ```
   ```
   WIREGUARD_PRIVATE_KEY=your-private-key-here
   ```
   ```bash
   chmod 600 ~/arr/.env ~/arr/docker-compose.yml
   ```
1. Create qBittorrent's config folder:
   ```bash
   mkdir -p ~/arr/qbittorrent
   ```
1. Add both services to `~/arr/docker-compose.yml`, alongside Sonarr, Radarr and Prowlarr. Replace `nordvpn` with your provider, and the countries with ones near you:
   ```yaml
     gluetun:
       image: qmcgaw/gluetun
       container_name: gluetun
       cap_add:
         - NET_ADMIN
       devices:
         - /dev/net/tun:/dev/net/tun
       ports:
         - 8080:8080         # qBittorrent web UI
         - 6881:6881         # qBittorrent incoming connections
         - 6881:6881/udp
       environment:
         - VPN_SERVICE_PROVIDER=nordvpn
         - VPN_TYPE=wireguard
         - WIREGUARD_PRIVATE_KEY=${WIREGUARD_PRIVATE_KEY}
         - SERVER_COUNTRIES=Netherlands,Germany,Sweden
         - TZ=America/Toronto
       labels:
         - com.centurylinklabs.watchtower.enable=false
       restart: unless-stopped

     qbittorrent:
       image: lscr.io/linuxserver/qbittorrent:latest
       container_name: qbittorrent
       network_mode: "service:gluetun"
       depends_on:
         - gluetun
       environment:
         - PUID=1000
         - PGID=1000
         - TZ=America/Toronto
         - WEBUI_PORT=8080
       volumes:
         - /home/pi/arr/qbittorrent:/config
         - /mnt/sda1:/data
       restart: unless-stopped
   ```
   - **No `ports:` on qbittorrent.** It has no network of its own to publish them on, and Compose refuses to start it if you add any. They go on gluetun instead.
   - `/mnt/sda1:/data` matches the other containers, so downloads land in `/data/downloads` and Sonarr/Radarr can hardlink them — see [Folder Layout](/Pi-Guide/Arr-Stack.md#folder-layout).
   - List **several** countries in `SERVER_COUNTRIES`. It gives Gluetun a pool of servers to choose from rather than one it keeps retrying.
   - If your provider needs an address, add `- WIREGUARD_ADDRESSES=10.5.0.2/32` (with the value it gave you).
   - The Watchtower label is explained in [Rules for Containers Sharing Gluetun's Network](#rules-for-containers-sharing-gluetuns-network).
   - **Port 8080 is already taken** if you moved [Pi-hole's web interface](/Pi-Guide/Pi-hole.md#changing-the-web-interface-port) there. Use e.g. `8081` instead, changing all three: both sides of `8080:8080` and `WEBUI_PORT`.
1. Start them:
   ```bash
   cd ~/arr
   docker compose up -d gluetun qbittorrent
   docker logs gluetun 2>&1 | tail -20
   ```
   Wait for Gluetun's log to show the VPN connected and a public IP. If it keeps restarting the connection, the key or address is wrong.
1. qBittorrent prints a temporary admin password on first start:
   ```bash
   docker logs qbittorrent 2>&1 | grep -i password
   ```
   Log in at `http://[PIIPADDRESS]:8080` with `admin` and that password, then set your own under Tools → Options → WebUI.
1. In qBittorrent's Tools → Options → Downloads, set the default save path to `/data/downloads`.

## Connect Sonarr and Radarr

Because everything is in one compose file, Sonarr and Radarr reach qBittorrent by name. Use `gluetun`, **not** `qbittorrent`: qBittorrent's network belongs to the gluetun container, so that's the name that answers.

In both Sonarr and Radarr: Settings → Download Clients → Add → qBittorrent:

- Host: `gluetun`
- Port: `8080`
- Username/password: the ones you just set

Click **Test**, then Save.

Using the container name also avoids a [firewall](/Pi-Guide/SSH-Hardening.md#firewall-ufw) problem. `http://[PIIPADDRESS]:8080` from another container counts as incoming traffic to the Pi, and UFW blocks it unless you add a rule.

## Testing

Check that qBittorrent's traffic leaves through the VPN and not your own connection:

```bash
curl -s https://ipinfo.io/ip; echo                                    # your home IP
docker exec gluetun wget -qO- https://ipinfo.io/ip; echo              # must be DIFFERENT
docker exec qbittorrent curl -s https://ipinfo.io/ip; echo            # must MATCH gluetun's
```

If the last two differ, qBittorrent isn't using Gluetun's network. Check `network_mode: "service:gluetun"` is set, then recreate both.

When the VPN is down, a command like the last one hangs or fails instead of printing your home IP. That's the kill switch working, not a fault. Some trackers and torrent sites also offer a "check my torrent IP" magnet link, which shows the address qBittorrent actually reports to peers.

## Rules for Containers Sharing Gluetun's Network

Getting these wrong leaves qBittorrent with no network. Nothing tells you why, except that downloads stall.

- **After restarting Gluetun, restart qBittorrent too.** `docker restart gluetun` keeps the same container, but qBittorrent's connection doesn't come back on its own:
  ```bash
  docker restart gluetun && docker restart qbittorrent
  ```
- **After recreating Gluetun, recreate qBittorrent in the same command.** Any change to Gluetun's settings or image recreates it, with a new container ID. qBittorrent is still attached to the old one and fails to start (`joining network namespace of container: No such container`):
  ```bash
  cd ~/arr
  docker compose up -d --force-recreate gluetun qbittorrent
  ```
- **Keep Watchtower away from Gluetun.** An automatic update is a recreate that doesn't include qBittorrent, which is why the compose file above has the `watchtower.enable=false` label. Update the pair by hand ([Updating](#updating)).
- **Anything else you add to Gluetun's network** (e.g. a second download client) follows the same rules: its ports go on gluetun, and it gets recreated alongside.

The published ports (8080, 6881) belong to a Docker container, so [they bypass UFW](/Pi-Guide/SSH-Hardening.md#firewall-ufw) like any other. Don't forward 8080 on your router. If you want the web UI limited to your LAN at the firewall level too, see [Restricting a Docker Port to Your LAN](/Pi-Guide/SSH-Hardening.md#restricting-a-docker-port-to-your-lan).

## Why WireGuard, Not OpenVPN

Gluetun supports both, and many older guides use OpenVPN. Prefer WireGuard:

- It connects in about a second, where OpenVPN takes several.
- **OpenVPN can get stuck on a dead server.** Gluetun reuses the last server's address when OpenVPN reconnects. If that server has gone down, it keeps retrying the same dead one. Over TCP each attempt takes about two minutes to time out. On one server this left the tunnel dead for eight minutes, and Gluetun's own health restart couldn't get out of it because it restarts OpenVPN with the same address.

If your provider only offers OpenVPN, use UDP (`OPENVPN_PROTOCOL=udp`, the default) rather than TCP. Put `OPENVPN_USER` and `OPENVPN_PASSWORD` in the `.env` file too, referenced as `${OPENVPN_USER}` / `${OPENVPN_PASSWORD}`. They're your provider's service credentials and deserve the same care as a WireGuard key.

## Updating

Always update the pair together:

```bash
cd ~/arr
docker compose pull gluetun qbittorrent
docker compose up -d --force-recreate gluetun qbittorrent
```

Then re-run the checks under [Testing](#testing).

## Sources

- https://github.com/qdm12/gluetun-wiki
- https://github.com/qdm12/gluetun-wiki/tree/main/setup/providers
- https://github.com/qdm12/gluetun-wiki/blob/main/setup/connect-a-container-to-gluetun.md
- https://docs.linuxserver.io/images/docker-qbittorrent/
