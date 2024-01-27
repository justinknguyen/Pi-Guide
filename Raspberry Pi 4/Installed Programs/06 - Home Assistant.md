# Home Assistant 
Can connect anything "smart" into one single app for a unified Smart Home. Main benefit is being able to integrate into Apple HomeKit.
## Table of Contents
- [Installation](#installation)
- [Testing](#testing)
- [Backing Up Home Assistant](#backing-up-home-assistant)
- [Sources](#sources)
## Installation
1. Create a `docker-compose.yml` file by typing:
    ```
    sudo nano docker-compose.yml
    ```
2. Inside the nano editor, copy and paste the following:
    ```
    version: '3'
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
    - Copying and pasting the above into the YAML file may not work due to formatting. Directly copy and paste it from here https://www.home-assistant.io/installation/raspberrypi/#docker-compose and replace `/PATH_TO_YOUR_CONFIG` with `/home/pi/homeassistant`
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
4. Install Home Assistant:
    ```
    docker-compose up -d
    ```
## Testing
You can now access the WebUI by typing `[PIIPADDRESS]:8123` into your search bar. Follow the second link under Sources to learn how to use Home Assistant.
## Backing Up Home Assistant
1. Stop the Home Assistant service by logging into Portainer.
2. Create a .tar file of the Home Assistant folder (mine is located at `/home/pi/homeassistant`):
    ```
    sudo tar -cf ha-backup.tar homeassistant/
    ```
3. Download WinSCP from https://winscp.net/eng/download.php. This is to easily transfer files between your computer and the Pi.
4. Transfer the `ha-backup.tar` file to your computer using WinSCP.
5. You can start up Home Assistant again with Portainer.
<!-- -->

To restore from a backup:
1. On your new Pi install, use WinSCP to transfer the .tar file from your computer to the Pi `/home/pi` directory, then unpack the .tar with (ensure Home Assistant is not running on the new Pi):
    ```
    sudo tar -xvf ha-backup.tar
    ```
2. Extracting the .tar should automatically overwrite the Home Assistant config folder. Now, start Home Assistant back up.
## Sources
* https://www.home-assistant.io/installation/raspberrypi/
* https://www.home-assistant.io/getting-started/onboarding/
* https://www.youtube.com/watch?v=u8vOxg-SI74
