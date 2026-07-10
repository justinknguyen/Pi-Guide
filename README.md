# Pi-Guide

A collection of setup guides for self-hosted services and utilities on a Raspberry Pi, covering networking, home automation, media, and backup tools.

> **Disclaimer:** These guides are provided as-is, with no guarantee against misconfiguration or data loss. Each guide is based on established community references, linked under its own Sources section.

[Table of Contents](#table-of-contents)

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

## Table of Contents

Grouped by category. Indented guides depend on their parent guide — each guide's own Prerequisites section lists exactly what it needs.

**Core System Setup**
- [Unattended-Upgrades](/Pi-Guide/Unattended-Upgrades.md) — automatic security updates
- [Watchdog](/Pi-Guide/Watchdog.md) — auto-reboot on system hang
- [XRDP](/Pi-Guide/XRDP.md) — remote desktop access

**Containers & Monitoring**
- [Docker](/Pi-Guide/Docker.md)
- [Portainer](/Pi-Guide/Portainer.md) — Docker web UI
- [Grafana](/Pi-Guide/Grafana.md) — hardware metrics dashboard

**Networking & DNS**
- [Pi-Hole](/Pi-Guide/Pi-Hole.md) — network-wide ad blocking
  - [Unbound](/Pi-Guide/Unbound.md) — recursive DNS resolver
  - [Gravity Sync](/Pi-Guide/Gravity-Sync.md) — sync two Pi-holes (deprecated, see guide)
  - [keepalived](/Pi-Guide/keepalived.md) — Pi-hole failover
- [PiVPN](/Pi-Guide/PiVPN.md) — WireGuard VPN server
- [NGINX](/Pi-Guide/NGINX.md) — web server / reverse proxy
  - [GoAccess](/Pi-Guide/GoAccess.md) — NGINX log analytics

**Home Automation**
- [Home Assistant](/Pi-Guide/Home-Assistant.md)
  - [Room Assistant](/Pi-Guide/Room-Assistant.md) — BLE room presence detection
  - [Mosquitto](/Pi-Guide/Mosquitto.md) — MQTT broker
- [diyHue](/Pi-Guide/diyHue.md) — Philips Hue Bridge emulator
- [Hyperion](/Pi-Guide/Hyperion.md) or [HyperHDR](/Pi-Guide/HyperHDR.md) — ambient TV lighting

**Storage & Backup**
- [NAS](/Pi-Guide/NAS.md) — network-attached storage
- [immich](/Pi-Guide/immich.md) — self-hosted photo/video backup
- [raspiBackup](/Pi-Guide/raspiBackup.md) — scheduled system backups
- [Rclone](/Pi-Guide/Rclone.md) — cloud backup

**Media & Gaming**
- [RetroPie](/Pi-Guide/RetroPie.md) — retro game emulation

**Apple Ecosystem**
- [AltServer](/Pi-Guide/AltServer.md) — sideload apps to iOS without a computer

**Personal Finance**
- [Wealthsimple to Actual Budget Sync](/Pi-Guide/Wealthsimple-to-ActualBudget-Sync.md) — automated transaction import
