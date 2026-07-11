# diyHue

Turn the Raspberry Pi into another Hue Bridge for multiple entertainment areas/hue sync instances.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

If Pi-Hole is installed, diyHue needs port 80, so change Pi-Hole's web interface port first.

Pi-hole v6+ no longer uses lighttpd — it ships its own embedded web server, configured in `/etc/pihole/pihole.toml`. Change the port with:

```bash
sudo pihole-FTL --config webserver.port 8080
```

Then test and access Pi-Hole's webui using `[PIIPADDRESS]:8080`.

If you're still on Pi-hole v5 (which served its web interface with lighttpd), change the port there instead:

1. Go to:
   ```bash
   sudo nano /etc/lighttpd/lighttpd.conf
   ```
1. Change the line that says `server.port = 80` to `server.port = 8080`.
1. Go to:
   ```bash
   sudo nano /etc/lighttpd/external.conf
   ```
1. Enter the following in the file and save it:
   ```
   server.port := 8080
   ```
1. Restart the service:
   ```bash
   sudo service lighttpd restart
   ```
1. Test and access Pi-Hole's webui using `[PIIPADDRESS]:8080`.

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
- https://raspberrypi.stackexchange.com/questions/52090/how-do-i-change-pi-holes-url-path#:~:text=There%20are%20many%20ways%20that,port%20number%20such%20as%208080%20.
