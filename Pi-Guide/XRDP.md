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
   ```
   sudo raspi-config
   ```
1. Go to (1) System Options -> S5 Boot/Auto Login -> select "B4 Desktop Autologin".
1. Exit back to the terminal and run the following commands to install the desktop environment:
   ```
   sudo apt update
   sudo apt-get install raspberrypi-ui-mods xinit xserver-xorg
   ```
1. Reboot:
   ```
   sudo reboot
   ```

## Installation

1. Install XRDP:
   ```
   sudo apt install xrdp
   ```
2. When the installation process is complete, the XRDP service will automatically start. You can verify that XRDP is running by entering:
   ```
   systemctl show -p SubState --value xrdp
   ```
3. By default XRDP uses the `/etc/ssl/private/ssl-cert-snakeoil.key` file which is readable only by users that are members of the “ssl-cert” group. You’ll need to add the user that runs the XRDP server to the ssl-cert group by entering:
   ```
   sudo adduser pi ssl-cert
   ```
   - Replace "pi" with the name of your login username if you changed it.

## Testing

Type "rdp" into your Windows search bar and open "Remote Desktop Connection". Once opened, you can enter the IP address of the Pi to login and view the desktop.

## Troubleshooting

If you get a blue screen and cannot connect to the RDP, just create a second user by:

1. Entering:
   ```
   sudo adduser <username>
   ```
2. Choose and confirm password.
3. Hit enter for defaults.
4. Try RDP again with that login.
5. Add user to ssl-cert group:
   ```
   sudo adduser <username> ssl-cert
   ```

## Sources

- https://linuxize.com/post/how-to-install-xrdp-on-raspberry-pi/
- https://stackoverflow.com/questions/70146297/raspberry-pi-remote-desktop-connection-problem-giving-up
