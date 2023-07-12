# diyHue
Turn the Raspberry Pi into another Hue Bridge for multiple entertainment areas/hue sync instances.
## Prerequisites
If you have Pi-Hole installed, you will need to change the port as diyHue will now need to use the default port (80).
1. Go to:
    ```
    sudo nano /etc/lighttpd/lighttpd.conf
    ```
2. Change the line that says `server.port = 80` to `server.port = 8080`.
3. Go to:
    ```
    sudo nano /etc/lighttpd/external.conf
    ```
4. Enter the following in the file and save it:
    ```
    sudo nano /etc/lighttpd/external.conf
    ```
5. Restart the service:
    ```
    sudo service lighttpd restart
    ```
6. You can test and access Pi-Hole's webui using `[PIIPADDRESS]:8080`.
## Installation
1. Enter the below and replace the X's with the Pi's MAC address:
    ```
    docker run -d --name diyHue --restart=always --network=host -e MAC=XX:XX:XX:XX:XX:XX -v /mnt/hue-emulator/config:/opt/hue-emulator/config diyhue/core:latest
    ```
2. Check the container is running using Portainer.
## Configuration
1. Access the webui using `[PIIPADDRESS]`.
2. Open up the Hue App on your phone, and go to `Settings` > `My Hue System`, and then add your diyHue bridge. When it prompts to press the link button, go to the diyHue webui and click on `Link Button` tab and press Link App.
3. If the Hue App says the bridge needs to be updated, go back to the webui and click on `Bridge` tab, and from there, you can emulate the diyHue bridge's software version to trick the Hue App.
4. If the Hue App says the bridge cannot be found, try manually entering the Pi's IP Address to search for it.
5. Once the diyHue bridge is setup in the app, go back to the webui and click on `Hue Bridge` tab, and enter the official Hue Bridge's IP Address, then pair it.
6. Once paired, you can then click on the `Lights` tab and scan for lights. Once it's able to search for all of your lights connected to the official Hue Bridge, you can finally go back to the Hue App, and scan for lights to make your rooms/entertainment areas.
## Sources
* https://diyhue.readthedocs.io/en/latest/getting_started.html#docker-install
* https://diyhue.discourse.group/t/app-requires-update-of-hue-bridge-update-fails/233
* https://raspberrypi.stackexchange.com/questions/52090/how-do-i-change-pi-holes-url-path#:~:text=There%20are%20many%20ways%20that,port%20number%20such%20as%208080%20.
