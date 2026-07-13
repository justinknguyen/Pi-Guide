# Syncthing

Continuous, private file sync between your devices — like Dropbox, but peer-to-peer with no cloud in the middle. Great for keeping a folder mirrored between your computer, phone, and the Pi (which then makes an always-on copy the rest of your backups can pick up).

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Installation

1. Add the official Syncthing APT repository (the Raspberry Pi OS repo version tends to lag far behind):
   ```bash
   sudo mkdir -p /etc/apt/keyrings
   sudo curl -L -o /etc/apt/keyrings/syncthing-archive-keyring.gpg https://syncthing.net/release-key.gpg
   echo "deb [signed-by=/etc/apt/keyrings/syncthing-archive-keyring.gpg] https://apt.syncthing.net/ syncthing stable" | sudo tee /etc/apt/sources.list.d/syncthing.list
   ```
1. Install it:
   ```bash
   sudo apt update
   sudo apt install syncthing
   ```
1. Enable it to run at boot for your user (replace `pi` with your username):
   ```bash
   sudo systemctl enable --now syncthing@pi.service
   ```

## Configuration

By default the web GUI only listens on the Pi itself, so open it up to your LAN:

1. Edit the config (on older Syncthing versions the file is at `~/.config/syncthing/config.xml` instead):
   ```bash
   nano ~/.local/state/syncthing/config.xml
   ```
1. Inside the `<gui>` section, change the `<address>` line to:
   ```xml
   <address>0.0.0.0:8384</address>
   ```
1. Restart:
   ```bash
   sudo systemctl restart syncthing@pi.service
   ```
1. Open `[PIIPADDRESS]:8384` in your address bar. Set a GUI username/password when it prompts you (Actions > Settings > GUI).
1. Install Syncthing on your other devices (desktop apps, or the Möbius Sync/Syncthing-Fork apps on iOS/Android), then on the Pi click "Add Remote Device" — devices find each other by Device ID (shown under Actions > Show ID).
1. Share a folder: "Add Folder" on one device, then tick the other devices under the folder's Sharing tab and accept the prompt on each.

## Testing

Drop a file into the shared folder on one device and watch it appear on the others. The GUI's folder panel shows "Up to Date" when everything is synced.

## Sources

- https://apt.syncthing.net/
- https://docs.syncthing.net/users/autostart.html#using-systemd
- https://docs.syncthing.net/intro/getting-started.html
