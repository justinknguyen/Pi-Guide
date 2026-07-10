# Mosquitto

MQTT communication — lets Room Assistant (on one Pi) send data to Home Assistant (on another Pi). Install Mosquitto on the Pi running Home Assistant.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

- [Home Assistant](/Pi-Guide/Home%20Assistant.md)
- [Room Assistant](/Pi-Guide/Room%20Assistant.md)

## Installation

1. SSH into the Pi with Home Assistant and enter:
   ```bash
   sudo apt update
   sudo apt upgrade
   ```
1. Install Mosquitto:
   ```bash
   sudo apt install mosquitto mosquitto-clients
   ```
1. Check status:
   ```bash
   sudo systemctl status mosquitto
   ```

## Configuration

1. Create a username and password. This will be used for Home Assistant:
   ```bash
   mosquitto_passwd -c passwordfile [USERNAME]
   ```
1. Edit the config file:
   ```bash
   sudo nano /etc/mosquitto/mosquitto.conf
   ```
1. Add the following lines to the bottom. This will allow you to use your Pi's IP address when testing.
   ```
   listener 1883
   allow_anonymous true
   ```
1. Reboot:
   ```bash
   sudo reboot
   ```

## Testing

1. Enter the following:
   ```bash
   mosquitto_sub -h [PIIPADDRESS] -t "mqtt/pimylifeup"
   ```
1. Open up another SSH terminal into the Pi (keep the other one open still), and enter the following:
   ```bash
   mosquitto_pub -h [PIIPADDRESS] -t "mqtt/pimylifeup" -m "Hello world"
   ```
1. You should see `Hello World` in your original terminal.

## Sources

- https://pimylifeup.com/raspberry-pi-mosquitto-mqtt-server/
- http://www.steves-internet-guide.com/mqtt-username-password-example/
