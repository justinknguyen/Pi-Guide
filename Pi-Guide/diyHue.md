# diyHue

Turn the Raspberry Pi into another Hue Bridge for multiple entertainment areas/hue sync instances.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

If Pi-Hole is installed, diyHue needs port 80, so change Pi-Hole's web interface port first — see [Pi-Hole: Changing the Web Interface Port](/Pi-Guide/Pi-Hole.md#changing-the-web-interface-port).

## Installation

1. Enter the below and replace the X's with the Pi's MAC address:
   ```bash
   docker run -d --name diyHue --restart=always --network=host -e MAC=XX:XX:XX:XX:XX:XX -v /mnt/hue-emulator/config:/opt/hue-emulator/config diyhue/core:latest
   ```
1. Check the container is running using Portainer.

## Configuration

1. Access the webui using `[PIIPADDRESS]`.
1. Open the Hue App on your phone, go to `Settings` > `My Hue System`, and add your diyHue bridge. When it prompts you to press the link button, go to the diyHue webui, click the `Link Button` tab, and press Link App.
1. If the Hue App says the bridge needs to be updated, go back to the webui and click the `Bridge` tab. From there, you can emulate the diyHue bridge's software version to trick the Hue App.
1. If the Hue App says the bridge cannot be found, try manually entering the Pi's IP address to search for it.
1. Once the diyHue bridge is set up in the app, go back to the webui and click the `Hue Bridge` tab. Enter the official Hue Bridge's IP address, then pair it.
1. Once paired, click the `Lights` tab and scan for lights. When diyHue finds all your lights connected to the official Hue Bridge, go back to the Hue App and scan for lights to build your rooms/entertainment areas.

## Sources

- https://diyhue.readthedocs.io/en/latest/getting_started.html#docker-install
- https://diyhue.discourse.group/t/app-requires-update-of-hue-bridge-update-fails/233
