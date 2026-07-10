# XRDP

Remotely access your Raspberry Pi's desktop environment from another device.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

1. Enter:
   ```bash
   sudo raspi-config
   ```
1. Go to (1) System Options -> S5 Boot/Auto Login -> select "B4 Desktop Autologin".
1. Exit back to the terminal and run the following commands to install the desktop environment:
   ```bash
   sudo apt update
   sudo apt-get install --no-install-recommends rpd-x-core rpd-x-extras xinit xserver-xorg
   ```
   - On Raspberry Pi OS "Trixie" (Debian 13) and newer, `raspberrypi-ui-mods` no longer exists — it was replaced by the modular `rpd-*` meta-packages. `rpd-x-core` (not `rpd-wayland-core`) is used here since XRDP/xorgxrdp needs an X11 session. If you're still on "Bookworm" (Debian 12) or earlier, `raspberrypi-ui-mods` is still valid.
1. Reboot:
   ```bash
   sudo reboot
   ```

## Installation

1. Install XRDP:
   ```bash
   sudo apt install xrdp
   ```
1. When the installation process is complete, the XRDP service will automatically start. You can verify that XRDP is running by entering:
   ```bash
   systemctl show -p SubState --value xrdp
   ```
1. XRDP's default `/etc/ssl/private/ssl-cert-snakeoil.key` file is only readable by members of the "ssl-cert" group, so add the user running the XRDP server to that group:
   ```bash
   sudo adduser pi ssl-cert
   ```
   - Replace "pi" with the name of your login username if you changed it.

## Testing

Type "rdp" into your Windows search bar and open "Remote Desktop Connection". Once opened, you can enter the IP address of the Pi to log in and view the desktop.

## Troubleshooting

If you get a blue screen and cannot connect to the RDP, create a second user:

1. Enter:
   ```bash
   sudo adduser <username>
   ```
1. Choose and confirm password.
1. Hit enter for defaults.
1. Try RDP again with that login.
1. Add user to ssl-cert group:
   ```bash
   sudo adduser <username> ssl-cert
   ```

## Sources

- https://linuxize.com/post/how-to-install-xrdp-on-raspberry-pi/
- https://stackoverflow.com/questions/70146297/raspberry-pi-remote-desktop-connection-problem-giving-up
