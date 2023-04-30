# Mosquitto 
MQTT communication. This is how the Pi Zero 2 W can communicate with the Pi 4 and provide Room Assistant data (on Pi Zero 2 W) to Home Assistant (on Pi 4). <br><br>
**Disclaimer** <br>
This will NOT be installed on the Pi Zero 2 W, instead it will be installed on the Pi 4. This guide is placed here for convenience with setting up with Room Assistant.
## Installation
1. SSH into Pi 4 and enter:
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
1. On the Pi 4, enter the following:
    ```
    mosquitto_sub -h [PIIPADDRESS] -t "mqtt/pimylifeup"
    ```
2. Open up another SSH terminal into the Pi 4 (keep the other one open still), and enter the following:
    ```
    mosquitto_pub -h [PIIPADDRESS] -t "mqtt/pimylifeup" -m "Hello world"
    ```
3. You should see `Hello World` in your original terminal.
## Sources
* https://pimylifeup.com/raspberry-pi-mosquitto-mqtt-server/
* http://www.steves-internet-guide.com/mqtt-username-password-example/