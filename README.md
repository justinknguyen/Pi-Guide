# Pi-Guide

A collection of setup guides for self-hosted services and utilities on a Raspberry Pi, covering networking, home automation, media, and backup tools.

> **Disclaimer:** These guides are provided as-is, with no guarantee against misconfiguration or data loss. Each guide is based on established community references, linked under its own Sources section.

[Table of Contents](#table-of-contents)

## Getting Started

### Setting up the Raspberry Pi

Download and install the `Raspberry Pi Imager` from https://www.raspberrypi.com/software/

Insert your micro SD card, open Raspberry Pi Imager, and select your Pi model and OS. "Raspberry Pi OS Lite (64-bit)" is recommended for headless setups that won't use a desktop environment.

After selecting your storage device and clicking "Next", a popup asks if you want to apply custom settings. Enter your WiFi credentials and enable SSH under the "Services" tab. You can set your password and hostname here, or later via [How to Change Your Password and Hostname](#how-to-change-your-password-and-hostname).

Once flashing finishes, insert the micro SD into your Pi and power it on.

### Connecting to the Raspberry Pi

To connect to your Pi, use an SSH client — `PuTTY` (https://www.putty.org/) on Windows, or `Termius` from the app store on Mac.

Connect to your Pi by entering its IP address or hostname (default hostname is `raspberrypi`), then enter the username and password you set during imaging in the "Setting up the Raspberry Pi" step above (Raspberry Pi OS hasn't shipped a default `pi`/`raspberry` login since April 2022).

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

Listed in recommended install order.

- [Unattended-Upgrades](/Pi-Guide/Unattended-Upgrades.md)
- [Watchdog](/Pi-Guide/Watchdog.md)
- [XRDP](/Pi-Guide/XRDP.md)
- [Rclone](/Pi-Guide/Rclone.md)
- [Docker](/Pi-Guide/Docker.md)
- [Portainer](/Pi-Guide/Portainer.md)
- [Grafana](/Pi-Guide/Grafana.md)
- [Home Assistant](/Pi-Guide/Home%20Assistant.md)
  - [Room Assistant](/Pi-Guide/Room%20Assistant.md)
  - [Mosquitto](/Pi-Guide/Mosquitto.md)
- [Pi-Hole](/Pi-Guide/Pi-Hole.md)
  - [Unbound](/Pi-Guide/Unbound.md)
  - [Gravity Sync](/Pi-Guide/Gravity%20Sync.md)
  - [keepalived](/Pi-Guide/keepalived.md)
- [PiVPN](/Pi-Guide/PiVPN.md)
- [NGINX](/Pi-Guide/NGINX.md)
  - [GoAccess](/Pi-Guide/GoAccess.md)
- [diyHue](/Pi-Guide/diyHue.md)
- [Hyperion](/Pi-Guide/Hyperion.md) or [HyperHDR](/Pi-Guide/HyperHDR.md)
- [AltServer](/Pi-Guide/AltServer.md)
- [NAS](/Pi-Guide/NAS.md)
- [immich](/Pi-Guide/immich.md)
- [RetroPie](/Pi-Guide/RetroPie.md)
- [raspiBackup](/Pi-Guide/raspiBackup.md)
- [Wealthsimple to Actual Budget Sync](/Pi-Guide/Wealthsimple-to-ActualBudget-Sync.md)
