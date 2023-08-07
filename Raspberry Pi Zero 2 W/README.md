# Table of Contents
* [Getting Started](#Getting-Started)
  * [Building the Raspberry Pi Zero 2 W](#Building-the-Raspberry-Pi-Zero-2-W)
  * [Setting up the Raspberry Pi Zero 2 W](#Setting-up-the-Raspberry-Pi-Zero-2-W) 
* [Installed Programs](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/)
  * [Unattended-Upgrades](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/01%20-%20Unattended-Upgrades.md)
  * [Watchdog](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/02%20-%20Watchdog.md)
  * [Docker](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/03%20-%20Docker.md)
  * [Portainer](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/04%20-%20Portainer.md)
  * [Mosquitto](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/05%20-%20Mosquitto.md)
  * [Pi-Hole, Gravity Sync, and keepalived](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/06%20-%20Pi-Hole%2C%20Gravity%20Sync%2C%20and%20keepalived.md)
  * [Hyperion](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/07%20-%20Hyperion)
  * [diyHue](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/08%20-%20diyHue.md)
  * [HyperHDR](/Raspberry%20Pi%20Zero%202%20W/Installed%20Programs/09%20-%20HyperHDR.md)
* [Uninstalled Programs](/Raspberry%20Pi%20Zero%202%20W/Uninstalled%20Programs/)
  * [Room Assistant](/Raspberry%20Pi%20Zero%202%20W/Uninstalled%20Programs/01%20-%20Room%20Assistant.md)

***

# Getting Started
The following model was purchased from CanaKit: https://www.canakit.com/raspberry-pi-zero-2-w.html
* Raspberry Pi Zero 2 W Board
* 64 GB Samsung EVO+ MicroSD Card
* 2.5A Power Supply
* Self cooling Aluminum Pi Zero Case
* USB OTG Cable
* Mini HDMI Adapter
## Building the Raspberry Pi Zero 2 W
A good YouTube video for building and setting it up: https://www.youtube.com/watch?v=NjQtP0NVcJ4 <br><br>
If you try connecting the included Micro HDMI adapter with the Flirc case on, you'll notice it won't fit. Simply shave down the outer trim of the adapter until it fits.

## Setting up the Raspberry Pi Zero 2 W
### Option 1: Setup Headless (Recommended)
If you want to setup the Pi headless (via SSH terminal instead of a monitor and Pi desktop), in the Raspberry Pi Imager, choose "Raspberry Pi OS Lite (64-bit)". Then in the Advanced options page, enable SSH and Internet.

You could also manually enable SSH and Internet. This website provides a good guide on how to enable SSH and Internet headless https://pimylifeup.com/headless-raspberry-pi-setup/. <br><br>
You can download the required files for SSH and Internet here:
* [ssh](https://github.com/justinknguyen/PiGuide/blob/349dbb43f6d59b7d5426713397d484182c751744/ssh) <br>
* [wpa_supplicant.conf](https://github.com/justinknguyen/PiGuide/blob/349dbb43f6d59b7d5426713397d484182c751744/wpa_supplicant.conf) 

**Recommendations**<br>
You should set a static IP address for the Pi in your router settings so it doesn’t change. Having the IP address static is essential to keep all your programs working. <br><br>

### Option 2: Setup with Monitor
Use your computer to install Raspberry Pi OS directly, download the Raspberry Pi Imager from https://www.raspberrypi.com/software/. In the Imager, choose "Raspberry Pi OS (64-bit)". <br>

Once imaged onto the MicroSD card, insert it in the Pi and then connect your mouse, keyboard, and HDMI cable to continue setup using a monitor and the Pi desktop. Once done, enable SSH under `Preferences > Raspberry Pi Configuration > Interfaces`. <br><br>
**Recommendations**<br>
You should set a static IP address for the Pi in your router settings so it doesn’t change. Having the IP address static is essential to keep all your programs working. <br><br>
**Troubleshooting** <br>
In rare cases, connecting to the monitor won't display anything. If so, try [Setup Headless](#Setup-Headless) or follow this website https://windowsreport.com/raspberry-pi-hdmi-not-working/ by uncommenting the following lines in `boot/config.txt` of the MicroSD card:
* hdmi_force_hotplug=1
* hdmi_drive=2 
<br><br>
<!-- -->

***********
Once you enable SSH through either of the above methods, download a terminal to SSH into the Pi, such as PuTTY https://www.putty.org/. This is how we’ll manage the Raspberry Pi from now on instead of using the Pi desktop.
***********

### Optional: Change Password
If you setup the Pi with a monitor or headless with the Advanced options, you would have been presented with the option to change your password. If you setup the Pi headless manually, you will need to SSH into the Pi using PuTTY with the Pi's IP address. The IP address of the Pi can be found under your router settings. <br><br>
The default username is `pi` and the password is `raspberry`. <br>
Once you SSH in, type the following to change your password:
`passwd`
<br><br>

### Optional: Change Hostname
This could’ve been set in the Advanced options page of the Imager. The steps to change the hostname after the fact is listed below.
1. SSH into Pi, and open the following file by entering:
    ```
    sudo nano /etc/hosts
    ```
2. At the bottom, change `raspberrypi` to whatever name you want for the Pi.
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
4. Next, open another file by entering:
    ```
    sudo nano /etc/hostname
    ```
5. Change `raspberrypi` to whatever name you want for the Pi.
6. To save the file, press `Ctrl+X` then `Y` then `Enter`.
7. Reboot:
    ```
    sudo reboot
    ```
