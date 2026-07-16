# Pi-Guide

A collection of setup guides for self-hosted services and utilities on a Raspberry Pi, covering networking, home automation, media, and backup tools.

> **Disclaimer:** These guides are provided as-is, with no guarantee against misconfiguration or data loss. Each guide is based on established community references, linked under its own Sources section.

## Getting Started

### Setting up the Raspberry Pi

1. Download and install the `Raspberry Pi Imager` from https://www.raspberrypi.com/software/
1. Insert your micro SD card, open Raspberry Pi Imager, and select your Pi model and OS. "Raspberry Pi OS Lite (64-bit)" is recommended for headless setups that won't use a desktop environment.
1. Select your storage device and click "Next". When the popup asks if you want to apply custom settings, enter your WiFi credentials and enable SSH under the "Services" tab.
   - You can set your password and hostname here, or later via [How to Change Your Password and Hostname](#how-to-change-your-password-and-hostname).
1. Once flashing finishes, insert the micro SD into your Pi and power it on.

### Connecting to the Raspberry Pi

1. Use an SSH client — `PuTTY` (https://www.putty.org/) on Windows, or `Termius` from the app store on Mac.
1. Connect to your Pi by entering its IP address or hostname (default is `raspberrypi`), then enter the username and password you set during imaging above.
   - Raspberry Pi OS hasn't shipped a default `pi`/`raspberry` login since April 2022.

### Give the Pi a Static IP

The guides in this repo reference the Pi by its IP address, and services like Pi-hole break if that address changes. Pick one:

- **Recommended:** set a DHCP reservation in your router's settings (usually under LAN > DHCP Server) binding the Pi's MAC address to a fixed IP. Nothing to configure on the Pi itself.
- Or set a static IP on the Pi. Raspberry Pi OS "Bookworm" and newer use NetworkManager — run `sudo nmtui`, select "Edit a connection", set IPv4 to Manual, and enter an address outside your router's DHCP range.

### How to Change Your Password and Hostname

#### Change Password

SSH into the Pi and enter the following to change your password:

```bash
sudo passwd
```

#### Change Hostname

1. SSH into the Pi, and open the hosts file by entering:
   ```bash
   sudo nano /etc/hosts
   ```
1. At the bottom, change `raspberrypi` to whatever name you want for the Pi.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Next, open the hostname file by entering:
   ```bash
   sudo nano /etc/hostname
   ```
1. Change `raspberrypi` to whatever name you want for the Pi.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Reboot:
   ```bash
   sudo reboot
   ```

### Starting Over (Factory Reset)

There's no software reset button on a Pi — wiping the SD card and reflashing is the actual fresh start. The good news is that's less work than it sounds:

- Raspberry Pi Imager already formats the card as part of writing the OS — you don't need a separate formatting step. Just power off the Pi, remove the card, and repeat [Setting up the Raspberry Pi](#setting-up-the-raspberry-pi) above from a computer.
- If you only want to wipe the card (no new OS), select "Erase" from Imager's OS list instead of an operating system.
- Imager remembers your last-used WiFi/SSH/username settings (gear icon, or `Ctrl+Shift+X`), so you're not retyping everything each time.
- If you set up a DHCP reservation per [Give the Pi a Static IP](#give-the-pi-a-static-ip), that lives on your router, not the SD card — it survives a reset and the Pi will get the same address again automatically.

**Faster, if you do this often:** reflashing still means redoing every setup step from scratch — WiFi, SSH, hostname, and all your guides. Skip that by keeping a clean backup instead. Right after your first full setup (before installing anything else), take a backup with [raspiBackup](/Pi-Guide/raspiBackup.md). Next time you want to start over, [restore that backup](/Pi-Guide/raspiBackup.md#restoring-the-whole-pi) onto the card instead of reflashing — it comes back exactly as it was the moment you backed it up, in a fraction of the time.

## Table of Contents

Grouped by category. Indented guides depend on their parent guide — each guide's own Prerequisites section lists exactly what it needs.

**Core System Setup**
- [SSH Hardening](/Pi-Guide/SSH-Hardening.md) — key-based login, firewall, fail2ban
- [Unattended-Upgrades](/Pi-Guide/Unattended-Upgrades.md) — automatic security updates
- [Log2Ram](/Pi-Guide/Log2RAM.md) — reduce SD card wear
- [Watchdog](/Pi-Guide/Watchdog.md) — auto-reboot on system hang
- [XRDP](/Pi-Guide/XRDP.md) — remote desktop access

**Containers & Monitoring**
- [Docker](/Pi-Guide/Docker.md)
- [Portainer](/Pi-Guide/Portainer.md) — Docker web UI
- [Watchtower](/Pi-Guide/Watchtower.md) — auto-update Docker containers
- [Grafana](/Pi-Guide/Grafana.md) — hardware metrics dashboard
- [Uptime Kuma](/Pi-Guide/Uptime-Kuma.md) — service status page and alerts
- [Homepage](/Pi-Guide/Homepage.md) — dashboard linking all your services

**Networking & DNS**
- [Pi-hole](/Pi-Guide/Pi-hole.md) — network-wide ad blocking
  - [Unbound](/Pi-Guide/Unbound.md) — recursive DNS resolver
  - [nebula-sync](/Pi-Guide/Nebula-Sync.md) — sync two Pi-holes
  - [keepalived](/Pi-Guide/keepalived.md) — Pi-hole failover
- [PiVPN](/Pi-Guide/PiVPN.md) — WireGuard VPN server
- [Tailscale](/Pi-Guide/Tailscale.md) — remote access without port forwarding
- [DDNS](/Pi-Guide/DDNS.md) — a hostname that follows your home IP (DuckDNS)
- [NGINX](/Pi-Guide/NGINX.md) — web server / reverse proxy
  - [GoAccess](/Pi-Guide/GoAccess.md) — NGINX log analytics

**Home Automation**
- [Home Assistant](/Pi-Guide/Home-Assistant.md)
  - [Room Assistant](/Pi-Guide/Room-Assistant.md) — BLE room presence detection
  - [Mosquitto](/Pi-Guide/Mosquitto.md) — MQTT broker
  - [Zigbee2MQTT](/Pi-Guide/Zigbee2MQTT.md) — Zigbee devices without brand hubs
- [diyHue](/Pi-Guide/diyHue.md) — Philips Hue Bridge emulator
- [Hyperion](/Pi-Guide/Hyperion.md) or [HyperHDR](/Pi-Guide/HyperHDR.md) — ambient TV lighting

**Storage & Backup**
- [NAS](/Pi-Guide/NAS.md) — network-attached storage
- [immich](/Pi-Guide/immich.md) — self-hosted photo/video backup
- [Syncthing](/Pi-Guide/Syncthing.md) — sync files between your devices
- [raspiBackup](/Pi-Guide/raspiBackup.md) — scheduled system backups
- [Rclone](/Pi-Guide/Rclone.md) — cloud backup

**Media & Gaming**
- [Jellyfin](/Pi-Guide/Jellyfin.md) — media streaming server
- [RetroPie](/Pi-Guide/RetroPie.md) — retro game emulation

**Security & Passwords**
- [Vaultwarden](/Pi-Guide/Vaultwarden.md) — self-hosted Bitwarden password manager

**Apple Ecosystem**
- [AltServer](/Pi-Guide/AltServer.md) — sideload apps to iOS without a computer
- [Scrypted](/Pi-Guide/Scrypted.md) — bring any camera into HomeKit

**Personal Finance**
- [Actual Budget](/Pi-Guide/Actual-Budget.md) — self-hosted budgeting app
- [Wealthsimple to Actual Budget Sync](/Pi-Guide/Wealthsimple-to-ActualBudget-Sync.md) — automated transaction import
