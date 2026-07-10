# Room Assistant

Uses Bluetooth Low Energy (BLE) to detect how far away a BLE-compatible device is from the Pi. This can be used to detect room presence and integrate with Home Assistant to set up smart home automations.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [Home Assistant](/Pi-Guide/Home%20Assistant.md)

## Installation

You may want to disable Pi-Hole temporarily before proceeding.

1. Create a new folder to store your room-assistant files in with:
   ```bash
   mkdir -p ~/room-assistant/config
   ```
1. Create a new config file with:
   ```bash
   nano ~/room-assistant/config/local.yml
   ```
1. Paste the following in with the proper replacements (click on the second link under Sources to learn how to use BLE and configure with your iPhone):
   ```yaml
   global:
     integrations:
       - homeAssistant
       - bluetoothClassic
   homeAssistant:
     mqttUrl: 'mqtt://homeassistant.local:1883'
     mqttOptions:
       username: youruser
       password: yourpass
   bluetoothClassic:
     addresses:
       - <bluetooth-mac-of-device-to-track>
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Create the docker-compose file with:
   ```bash
   nano ~/room-assistant/docker-compose.yml
   ```
1. Paste the following in and save:
   ```yaml
   services:
     room-assistant:
       image: mkerix/room-assistant
       restart: unless-stopped
       network_mode: host
       privileged: true
       volumes:
         - /var/run/dbus:/var/run/dbus
         - /home/pi/room-assistant/config:/room-assistant/config
   ```
   - Optional: install beta version by editing `docker-compose.yml` to:
     ```yaml
     services:
       room-assistant:
         image: mkerix/room-assistant:beta
         restart: unless-stopped
         network_mode: host
         privileged: true
         volumes:
           - /var/run/dbus:/var/run/dbus
           - /home/pi/room-assistant/config:/room-assistant/config
     ```
1. cd into the room-assistant directory:
   ```bash
   cd ~/room-assistant
   ```
1. Install Room Assistant:
   ```bash
   docker compose up
   ```
1. Check Portainer if Room Assistant is running correctly.

## Configuration

These steps will show you how to set up Room Assistant (on one Pi) with Home Assistant (on another Pi).

Head to [Mosquitto](/Pi-Guide/Mosquitto.md) and complete the steps. After completing, log in to the WebUI of Home Assistant and head over to Configuration>Devices & Services. Next, add the "MQTT" integration and set the following fields to:

- Broker: [PIIPADDRESS]
  - IP address of Pi 4.
- Port: 1883
- Username: [USERNAME]
  - What you created while setting up Mosquitto.
- Password: [PASSWORD]
  - What you created while setting up Mosquitto.
    <!-- -->
    Within your Room Assistant config file (Step 3 of Configuration), replace `homeassistant.local` with the IP address of Pi 4. <br><br>
    Finally, follow this YouTube video to learn how to set up simple cards and automations in Home Assistant with Room Assistant: https://www.youtube.com/watch?v=x5ublCxDDWE&t=379s.

## Troubleshooting

Sometimes Room Assistant's Bluetooth communication stops working (perhaps due to updating/reinstalling NodeJS). Run the following command to fix it:

```bash
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
```

You may also rerun the following commands:

```bash
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
sudo setcap cap_net_raw+eip $(eval readlink -f `which hcitool`)
sudo setcap cap_net_admin+eip $(eval readlink -f `which hciconfig`)
```

Restart the Pi once finished.

## Sources

- https://www.room-assistant.io/guide/quickstart-docker.html
- https://www.room-assistant.io/guide/quickstart-pi.html#configuring-room-assistant
- https://www.room-assistant.io/integrations/bluetooth-low-energy.html#settings
- https://www.youtube.com/watch?v=x5ublCxDDWE&t=379s
