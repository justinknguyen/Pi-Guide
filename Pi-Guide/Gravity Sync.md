# Gravity Sync

Sync Pi-Hole adlists and whitelists between two Raspbery Pi's.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- [Pi-Hole](/Pi-Guide/Pi-Hole.md)
- [Unbound](/Pi-Guide/Unbound.md) (recommended)

## Installation

This install guide will begin by setting up syncing between the two Pi's. Adding adlists or whitelists in one Pi will automatically add them to the other Pi.

1. Match your settings in Pi-Hole in your second Pi with your primary Pi.
2. Disable DHCP in Pi-Hole settings.
3. SSH into second Pi and enter:
   ```
   sudo apt update && sudo apt install sqlite3 sudo git rsync ssh
   ```
4. Ensure passwordless sudo. On secondary Pi, enter:
   ```
   sudo EDITOR=nano visudo
   ```
5. Scroll down to where you see `%sudo ALL=(ALL:ALL) ALL` and comment it out with `#`. Enter the following underneath it:
   ```
   %sudo ALL=(ALL:ALL) NOPASSWD:ALL
   ```
6. To save the file, press `Ctrl+X` then `Y` then `Enter`.
7. Reboot:
   ```
   sudo reboot
   ```
8. Repeat Steps 3-7 on your primary Pi.
9. On both Pi's, enter:
   ```
   curl -sSL https://raw.githubusercontent.com/vmstan/gs-install/main/gs-install.sh | bash
   ```
10. You will receive prompts to enter an IP Address and host username. This is NOT the host name of the Pi, but the username you use to log in. 
   - On the primary Pi, enter the secondary Pi's IP Address and host username (pi), if it says the authenticity of the host can't be established, enter `yes` in the following prompt. 
   - On the secondary Pi, enter the primary Pi's IP Address and host username (pi).

## Configuration

1. Check if there are any differences between the two Pi's in Pi-Hole by entering in the second Pi:
   ```
   gravity-sync compare
   ```
2. Pull settings from primary Pi to secondary Pi by entering in the second Pi:
   ```
   gravity-sync pull
   ```
3. You should now see adlists and whitelists synchronised between the two Pi's.
4. Next is to automate this syncing between the two Pi's on a schedule:
   ```
   gravity-sync auto
   ```
5. Optional: set sync frequency to 15 mins.
   ```
   gravity-sync auto quad
   ```

## Troubleshooting

- If you messed up the configuration, enter the following to restart the config:
  ```
  gravity-sync config
  ```
- If you need to completely restart, remove Gravity Sync by entering the following:
  ```
  gravity-sync purge
  ```
- If you are getting the error "sh: 1: cannot create .ssh/authorized_keys: Permission denied" after entering the remote Pi's username and password, you will need to enter the following on the remote Pi:
  ```
  chmod 700 .ssh
  sudo chmod 640 .ssh/authorized_keys
  sudo chown pi .ssh
  sudo chown pi .ssh/authorized_keys
  ```
  - replace "pi" with the Pi's username.
- If pushing and pulling is failing, check if the rsync versions are the same between both Pi's:
  ```
  rsync --version
  ```
  1. If not, it's likely one is on v3.2.3, and in order to update to the latest (v3.2.7 currently), install dependencies:
     ```
     sudo apt install gcc g++ gawk autoconf automake python3-cmarkgfm libssl-dev attr libxxhash-dev libattr1-dev liblz4-dev libzstd-dev acl libacl1-dev -y
     ```
  1. Download the latest rsync file:
     ```
     wget https://download.samba.org/pub/rsync/src/rsync-3.2.7.tar.gz
     ```
  1. Extract the file:
     ```
     tar -xf rsync-3.2.7.tar.gz
     ```
  1. Configure rsync:
     ```
     cd rsync-3.2.7
     ./configure
     ```
  1. Prepare rsync install files:
     ```
     make
     ```
  1. Install rsync:
     ```
     sudo make install
     ```
  1. Reboot and confirm the rsync version:
     ```
     sudo reboot
     ```

## Sources

- https://www.youtube.com/watch?v=IFVYe3riDRA
- https://github.com/vmstan/gravity-sync
- https://linuxhint.com/update-rsync-raspberry-pi/
