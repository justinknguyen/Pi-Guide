# Mosquitto

MQTT communication. This is how a one Pi can communicate with another Pi to provide Room Assistant (on one Pi) data to Home Assistant (on another Pi). <br><br>
**Disclaimer** <br>
This will NOT be installed on the Pi Zero 2 W, instead it will be installed on the Pi 4. This guide is placed here for convenience with setting up with Room Assistant.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Installation

1. SSH into the Pi with Home Assistant and enter:
   ```
   sudo apt update
   sudo apt upgrade
   ```
2. Install Mosquitto:
   ```
   sudo apt install mosquitto mosquitto-clients
   ```
3. Check status:
   ```
   sudo systemctl status mosquitto
   ```

## Configuration

1. Create a username and password. This will be used for Home Assistant:
   ```
   mosquitto_passwd -c passwordfile [USERNAME]
   ```
2. Edit the config file:
   ```
   sudo nano /etc/mosquitto/mosquitto.conf
   ```
3. Add the following lines to the bottom. This will allow you to use your Pi's IP address when testing.
   ```
   listener 1883
   allow_anonymous true
   ```
4. Reboot:
   ```
   sudo reboot
   ```

## Testing

1. Enter the following:
   ```
   mosquitto_sub -h [PIIPADDRESS] -t "mqtt/pimylifeup"
   ```
2. Open up another SSH terminal into the Pi (keep the other one open still), and enter the following:
   ```
   mosquitto_pub -h [PIIPADDRESS] -t "mqtt/pimylifeup" -m "Hello world"
   ```
3. You should see `Hello World` in your original terminal.

## Sources

- https://pimylifeup.com/raspberry-pi-mosquitto-mqtt-server/
- http://www.steves-internet-guide.com/mqtt-username-password-example/
