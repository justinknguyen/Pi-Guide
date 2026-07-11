# Home Assistant

Connects any "smart" device into a single app for a unified Smart Home. Main benefit: integrates with Apple HomeKit.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Backing Up Home Assistant](#backing-up-home-assistant)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

1. Create a `docker-compose.yml` file by entering:
   ```bash
   sudo nano docker-compose.yml
   ```
1. Inside the nano editor, copy and paste the following:
   ```yaml
   services:
     homeassistant:
       container_name: homeassistant
       image: "ghcr.io/home-assistant/home-assistant:stable"
       volumes:
         - /home/pi/homeassistant:/config
         - /etc/localtime:/etc/localtime:ro
         - /run/dbus:/run/dbus:ro
       restart: unless-stopped
       privileged: true
       network_mode: host
   ```
   - This matches the official compose file from https://www.home-assistant.io/installation/raspberrypi/#docker-compose, with `/PATH_TO_YOUR_CONFIG` set to `/home/pi/homeassistant`. If pasting breaks the YAML indentation, copy it from there directly.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Install Home Assistant:
   ```bash
   docker compose up -d
   ```

## Testing

You can now access the WebUI by entering `[PIIPADDRESS]:8123` into your address bar. Follow the second link under Sources to learn how to use Home Assistant.

## Backing Up Home Assistant

1. Stop the Home Assistant service by logging into Portainer.
1. Create a .tar file of the Home Assistant folder (for example, located at `/home/pi/homeassistant`):
   ```bash
   sudo tar -cf ha-backup.tar homeassistant/
   ```
1. Download WinSCP from https://winscp.net/eng/download.php to transfer files between your computer and the Pi.
1. Transfer the `ha-backup.tar` file to your computer using WinSCP.
1. Start up Home Assistant again with Portainer.

To restore from a backup:

1. On your new Pi install, use WinSCP to transfer the .tar file from your computer to the Pi `/home/pi` directory, then unpack the .tar (make sure Home Assistant is not running on the new Pi) using the following command:
   ```bash
   sudo tar -xvf ha-backup.tar
   ```
1. Extracting the .tar should automatically overwrite the Home Assistant config folder. Now, start Home Assistant back up.

## Sources

- https://www.home-assistant.io/installation/raspberrypi/
- https://www.home-assistant.io/getting-started/onboarding/
- https://www.youtube.com/watch?v=u8vOxg-SI74
