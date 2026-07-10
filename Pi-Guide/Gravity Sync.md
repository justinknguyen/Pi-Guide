# Gravity Sync

Sync Pi-Hole adlists and whitelists between two Raspberry Pis.

> **⚠ Deprecated/incompatible with current Pi-hole:** Gravity Sync was officially retired by its maintainer on July 26, 2024, and its GitHub repo is now archived (read-only). Its final release (v4.0.7) only works with Pi-hole 5.x — it is **not compatible with Pi-hole 6+**, which is what the [Pi-Hole](/Pi-Guide/Pi-Hole.md) install guide in this repo installs today. Following this guide as written will not work on a fresh setup. For syncing config between two Pi-holes on Pi-hole 6+, look into Pi-hole's built-in Teleporter sync or a community alternative like [Orbital Sync](https://github.com/mattwebbio/orbital-sync) instead.

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

This sets up syncing between the two Pis — adding adlists or whitelists on one Pi will automatically add them to the other.

1. Match your settings in Pi-Hole in your second Pi with your primary Pi.
1. Disable DHCP in Pi-Hole settings.
1. SSH into second Pi and enter:
   ```bash
   sudo apt update && sudo apt install sqlite3 sudo git rsync ssh
   ```
1. Ensure passwordless sudo. On secondary Pi, enter:
   ```bash
   sudo EDITOR=nano visudo
   ```
1. Scroll down to where you see `%sudo ALL=(ALL:ALL) ALL` and comment it out with `#`. Enter the following underneath it:
   ```
   %sudo ALL=(ALL:ALL) NOPASSWD:ALL
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Reboot:
   ```bash
   sudo reboot
   ```
1. Repeat Steps 3-7 on your primary Pi.
1. On both Pis, enter:
   ```bash
   curl -sSL https://raw.githubusercontent.com/vmstan/gs-install/main/gs-install.sh | bash
   ```
1. You will receive prompts to enter an IP Address and host username. This is NOT the host name of the Pi, but the username you use to log in. 
    - On the primary Pi, enter the secondary Pi's IP Address and host username (pi), if it says the authenticity of the host can't be established, enter `yes` in the following prompt. 
    - On the secondary Pi, enter the primary Pi's IP Address and host username (pi).

## Configuration

1. Check if there are any differences between the two Pis in Pi-Hole by entering in the second Pi:
   ```bash
   gravity-sync compare
   ```
1. Pull settings from primary Pi to secondary Pi by entering in the second Pi:
   ```bash
   gravity-sync pull
   ```
1. You should now see adlists and whitelists synchronised between the two Pis.
1. Next, automate this syncing between the two Pis on a schedule:
   ```bash
   gravity-sync auto
   ```
1. Optional: set sync frequency to 15 mins.
   ```bash
   gravity-sync auto quad
   ```

## Troubleshooting

- If you messed up the configuration, enter the following to restart the config:
  ```bash
  gravity-sync config
  ```
- If you need to completely restart, remove Gravity Sync by entering the following:
  ```bash
  gravity-sync purge
  ```
- If you are getting the error "sh: 1: cannot create .ssh/authorized_keys: Permission denied" after entering the remote Pi's username and password, you will need to enter the following on the remote Pi:
  ```bash
  chmod 700 .ssh
  sudo chmod 640 .ssh/authorized_keys
  sudo chown pi .ssh
  sudo chown pi .ssh/authorized_keys
  ```
  - replace "pi" with the Pi's username.
- If pushing and pulling is failing, check if the rsync versions are the same between both Pis:
  ```bash
  rsync --version
  ```
  1. If not, it's likely one is on v3.2.3, and in order to update to the latest (v3.2.7 currently), install dependencies:
     ```bash
     sudo apt install gcc g++ gawk autoconf automake python3-cmarkgfm libssl-dev attr libxxhash-dev libattr1-dev liblz4-dev libzstd-dev acl libacl1-dev -y
     ```
  1. Download the latest rsync file:
     ```bash
     wget https://download.samba.org/pub/rsync/src/rsync-3.2.7.tar.gz
     ```
  1. Extract the file:
     ```bash
     tar -xf rsync-3.2.7.tar.gz
     ```
  1. Configure rsync:
     ```bash
     cd rsync-3.2.7
     ./configure
     ```
  1. Prepare rsync install files:
     ```bash
     make
     ```
  1. Install rsync:
     ```bash
     sudo make install
     ```
  1. Reboot and confirm the rsync version:
     ```bash
     sudo reboot
     ```

## Sources

- https://www.youtube.com/watch?v=IFVYe3riDRA
- https://github.com/vmstan/gravity-sync
- https://linuxhint.com/update-rsync-raspberry-pi/
