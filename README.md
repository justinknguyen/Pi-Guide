# Pi-Guide

This is a guide and collection of packages I've installed on my Raspberry Pi's.

I am not responsible for anything happening to your Raspberry Pi. All guides are based on pre-existing guides on the internet.

[Table of Contents](#table-of-contents)

## Getting Started

### Setting up the Raspberry Pi

Download and install the `Raspberry Pi Imager` from https://www.raspberrypi.com/software/

After installing, insert your micro SD card into your computer and open the Raspberry Pi Imager. Select your Raspberry Pi model and choose your preferred OS. I recommend "Raspberry Pi OS Lite (64-bit)" if you will not use the desktop environment.

After selecting your storage device and clicking on "Next", a popup will appear asking if you want to apply custom settings. In the settings, enter your WiFi credentials and enable SSH under the "Services" tab. You can set your password and hostname at this point, or set it later by following the steps under [How to Change Your Password and Hostname](#how-to-change-your-password-and-hostname).

Once the firmware is done flashing to your micro SD, insert it in your Pi and power it on.

### Connecting to the Raspberry Pi

To connect to your Pi and use it, download a terminal to SSH into the Pi, such as `PuTTY` https://www.putty.org/ for Windows users. For Mac users, you can download `Termius` from the app store.

Once your terminal app is installed, connect to your Pi by entering it's IP address or hostname; the default hostname is `raspberrypi`. Once connected, you'll be asked to enter your username and password; the default username is `pi` and the password is `raspberry`.

### How to Change Your Password and Hostname

#### Change Password

SSH into the Pi and enter the following to change your password:

```
sudo passwd
```

#### Change Hostname

1. SSH into the Pi, and open the hosts file by entering:
   ```
   sudo nano /etc/hosts
   ```
2. At the bottom, change `raspberrypi` to whatever name you want for the Pi.
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
4. Next, open the hostname file by entering:
   ```
   sudo nano /etc/hostname
   ```
5. Change `raspberrypi` to whatever name you want for the Pi.
6. To save the file, press `Ctrl+X` then `Y` then `Enter`.
7. Reboot:
   ```
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
- [raspiBackup](/Pi-Guide/raspiBackup.md)
