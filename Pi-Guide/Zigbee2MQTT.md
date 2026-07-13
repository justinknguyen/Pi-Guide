# Zigbee2MQTT

Use Zigbee devices (sensors, buttons, bulbs, plugs — IKEA, Aqara, Hue, Sonoff, and thousands more) without any brand hubs. Devices are published over MQTT and show up automatically in Home Assistant.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [Mosquitto](/Pi-Guide/Mosquitto.md)
- [Home Assistant](/Pi-Guide/Home-Assistant.md) (recommended)
- A Zigbee USB coordinator — the SONOFF Zigbee 3.0 USB Dongle Plus (ZBDongle-E or ZBDongle-P) is the common budget pick (~$25). Plug it in with a short USB extension cable; sitting directly in the Pi's USB port causes interference from USB 3.0.

## Installation

1. Plug in the coordinator and find its stable device path:
   ```bash
   ls -l /dev/serial/by-id/
   ```
   Note the full `usb-...` name — using the `by-id` path (instead of `/dev/ttyACM0`, which can change) survives reboots.
1. Create a folder and compose file:
   ```bash
   mkdir -p ~/zigbee2mqtt/data
   nano ~/zigbee2mqtt/docker-compose.yml
   ```
1. Paste the following in, replacing the device path with yours. (The web UI is mapped to port `8081` here since 8080 is used by Pi-hole's relocated web interface in this repo's guides):
   ```yaml
   services:
     zigbee2mqtt:
       container_name: zigbee2mqtt
       image: koenkk/zigbee2mqtt
       restart: unless-stopped
       volumes:
         - ./data:/app/data
         - /run/udev:/run/udev:ro
       ports:
         - 8081:8080
       environment:
         - TZ=America/Edmonton
       devices:
         - /dev/serial/by-id/[YOURADAPTERID]:/dev/ttyACM0
   ```
1. Create the Zigbee2MQTT config file:
   ```bash
   nano ~/zigbee2mqtt/data/configuration.yaml
   ```
1. Paste the following in, filling in the IP of the Pi running Mosquitto and the MQTT username/password you created in the Mosquitto guide:
   ```yaml
   homeassistant:
     enabled: true
   frontend:
     enabled: true
   mqtt:
     base_topic: zigbee2mqtt
     server: mqtt://[PIIPADDRESS]:1883
     user: [USERNAME]
     password: [PASSWORD]
   serial:
     port: /dev/ttyACM0
     adapter: ember
   ```
   - `adapter: ember` is for the ZBDongle-E. Use `adapter: zstack` for the ZBDongle-P. Other coordinators: check the supported adapters page under Sources.
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then start it:
   ```bash
   cd ~/zigbee2mqtt
   docker compose up -d
   ```

## Configuration

1. Open the web UI at `[PIIPADDRESS]:8081`.
1. Click "Permit join" (top bar), then put your Zigbee device in pairing mode (usually holding its reset/pair button) — it will appear in the device list within seconds. Rename devices to something meaningful.
1. If you use Home Assistant with the MQTT integration set up (see [Room Assistant - Configuration](/Pi-Guide/Room-Assistant.md#configuration) for adding the MQTT integration), every paired device appears in Home Assistant automatically.

## Testing

Trigger a paired device (press the button, trip the sensor) and watch its state update live in the Zigbee2MQTT web UI and in Home Assistant.

## Updating

```bash
cd ~/zigbee2mqtt
docker compose pull && docker compose up -d
```

## Sources

- https://www.zigbee2mqtt.io/guide/installation/02_docker.html
- https://www.zigbee2mqtt.io/guide/adapters/
- https://www.zigbee2mqtt.io/guide/configuration/
