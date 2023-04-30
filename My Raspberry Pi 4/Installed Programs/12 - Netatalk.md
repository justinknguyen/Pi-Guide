# Netatalk
Transfer files between your computer and Raspberry Pi.
## Installation
1. Install Netatalk:
    ```
    sudo apt-get install netatalk
    ```
2. Edit the config file by deleting the preceding semicolons `;` and replacing `/xxxx` with `/home`:
    ```
    sudo nano /etc/netatalk/afp.conf
    ```
    ```
    [Homes]
    basedir regex = /home
    ```
3. You can add a custom directory by adding similar lines as below:
    ```
    [HTML]
    path = /var/www/html/
    ```
4. Restart the service:
    ```
    sudo systemctl restart netatalk
    ```  
## Testing
For Mac's, open Finder and click on Network. There you should see your Raspberry Pi listed, connect to your pi with the same credentials you use to SSH in.
## Troubleshooting
If the directory you are trying to write to is protected, enter the following command:
```
sudo chmod -R 777 [YOUR PATH HERE]
```
If it doesn't seem like Netatalk is working, just restart the service:
```
sudo systemctl restart netatalk
```
## Sources
* https://gallaugher.com/makersnack-set-up-your-raspberry-pi-so-that-you-can-use-it-with-the-mac-finders-file-system/
* https://forums.raspberrypi.com/viewtopic.php?t=155067