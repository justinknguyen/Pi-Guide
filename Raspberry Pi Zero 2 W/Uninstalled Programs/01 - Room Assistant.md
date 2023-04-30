# Room Assistant
Using BluetoothLowEnergy (BLE) to detect the distance away from the Pi using a BLE compatible device. This can be used to detect room presence and integrate with Home Assistant to setup smart home automations.
## Installation
You may want to disable Pi-Hole temporarily before proceeding.
1. Create a new folder to store your room-assistant files in with:
    ```
    mkdir -p ~/room-assistant/config
    ```
2. Create a new config file with:
    ```
    nano ~/room-assistant/config/local.yml
    ```
3. Paste the following in with the proper replacements (click on the second link under Sources to learn how to use BLE and configure with your iPhone):
    ```
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
4. To save the file, press `Ctrl+X` then `Y` then `Enter`. You can now follow the steps outlined under [How to Setup Room Assistant (on Pi Zero 2 W) with Home Assistant (on Pi 4)](#How-to-Setup-Room-Assistant-on-Pi-Zero-2-W-with-Home-Assistant-on-Pi-4).
5. Create the docker-compose file with:
    ```
    nano ~/room-assistant/docker-compose.yml
    ```
6. Paste the following in and save: 
    ```
    version: '3'
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
7. cd into the room-assistant directory:
    ```
    cd ~/room-assistant
    ```
8. Install Room Assistant:
    ```
    docker-compose up
    ```
9. Check Portainer if Room Assistant is running correctly.
## How to Setup Room Assistant (on Pi Zero 2 W) with Home Assistant (on Pi 4)
Head to [Mosquitto](#Mosquitto) and complete the steps. After completing, login to the WebUI of Home Assistant and head over to Configuration>Devices & Services. Next, add the "MQTT" integration and set the following fields to:
* Broker: [PIIPADDRESS]
  * IP address of Pi 4.
* Port: 1883
* Username: [USERNAME]
  * What you created while setting up Mosquitto.
* Password: [PASSWORD]
  * What you created while setting up Mosquitto.
<!-- -->
Within your Room Assistant config file (Step 3 of Configuration), replace `homeassistant.local` with the IP address of Pi 4. <br><br>
Finally, follow this YouTube video to learn how to setup simple cards and automations in Home Assistant with Room Assistant: https://www.youtube.com/watch?v=x5ublCxDDWE&t=379s.
## Optional: Install Beta Version
1. Edit `docker-compose.yml` to:
    ```
    sudo nano ~/room-assistant/docker-compose.yml
    ```
    ```
    version: '3'
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
2. cd to where the `docker-compose.yml` is and enter:
    ```
    docker-compose stop && docker-compose up
    ```
## Troubleshooting
Sometimes Room Assistant bluetooth communication stops working (perhaps due to updating/reinstalling NodeJS). Run the following command to fix it:
```
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
```
you may also rerun the following commands:
```
sudo setcap cap_net_raw+eip $(eval readlink -f `which node`)
sudo setcap cap_net_raw+eip $(eval readlink -f `which hcitool`)
sudo setcap cap_net_admin+eip $(eval readlink -f `which hciconfig`)
```
Restart the Pi once finished.
## Sources
* https://www.room-assistant.io/guide/quickstart-docker.html
* https://www.room-assistant.io/guide/quickstart-pi.html#configuring-room-assistant
* https://www.room-assistant.io/integrations/bluetooth-low-energy.html#settings
* https://www.youtube.com/watch?v=x5ublCxDDWE&t=379s