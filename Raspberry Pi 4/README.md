# Table of Contents
* [Getting Started](#Getting-Started)
  * [Building the Raspberry Pi 4](#Building-the-Raspberry-Pi-4)
  * [Setting up the Raspberry Pi 4](#Setting-up-the-Raspberry-Pi-4)
* [Installed Programs](/Raspberry%20Pi%204/Installed%20Programs/)
  * [Unattended-Upgrades](/Raspberry%20Pi%204/Installed%20Programs/01%20-%20Unattended-Upgrades.md)
  * [Watchdog](/Raspberry%20Pi%204/Installed%20Programs/02%20-%20Watchdog.md)
  * [XRDP](/Raspberry%20Pi%204/Installed%20Programs/03%20-%20XRDP.md)
  * [Docker](/Raspberry%20Pi%204/Installed%20Programs/04%20-%20Docker.md)
  * [Portainer](/Raspberry%20Pi%204/Installed%20Programs/05%20-%20Portainer.md)
  * [Home Assistant](/Raspberry%20Pi%204/Installed%20Programs/06%20-%20Home%20Assistant.md)
  * [Pi-Hole](/Raspberry%20Pi%204/Installed%20Programs/07%20-%20Pi-Hole.md)
  * [Unbound](/Raspberry%20Pi%204/Installed%20Programs/08%20-%20Unbound.md)
  * [PiVPN](/Raspberry%20Pi%204/Installed%20Programs/09%20-%20PiVPN.md)
  * [NGINX](/Raspberry%20Pi%204/Installed%20Programs/10%20-%20NGINX.md)
  * [GoAccess](/Raspberry%20Pi%204/Installed%20Programs/11%20-%20GoAccess.md)
  * [Netatalk](/Raspberry%20Pi%204/Installed%20Programs/12%20-%20Netatalk.md)
  * [AltServer](/Raspberry%20Pi%204/Installed%20Programs/13%20-%20AltServer.md)
* [Uninstalled Programs](/Raspberry%20Pi%204/Uninstalled%20Programs/)
  * [Rclone](/Raspberry%20Pi%204/Uninstalled%20Programs/01%20-%20Rclone.md)
  * [Grafana](/Raspberry%20Pi%204/Uninstalled%20Programs/02%20-%20Grafana.md)
  
***

# Getting Started
The following model was purchased from CanaKit: https://www.canakit.com/raspberry-pi-4-starter-kit.html
* Raspberry Pi 4 Model B with 1.5GHz 64-bit quad-core ARMv8 CPU
* 4GB RAM
* CanaKit 3.5A USB-C Power Supply with Noise Filter (UL Listed) specially designed for the Raspberry Pi 4 (5-foot cable)
* CanaKit Premium Black Case (High Gloss), Set of 3 Aluminum Heat Sinks
* CanaKit Fan
* Micro HDMI to HDMI Cable (6-foot cable)
* 32 GB Samsung EVO+ MicroSD Card
* USB Card Reader Dongle

## Building the Raspberry Pi 4
A good YouTube video for building and setting it up: https://www.youtube.com/watch?v=7rcNjgVgc-I <br>
### Modifying the Case Fan
The provided case fan outputs too much noise running at 100% constantly. <br>

Installing a 30 Ohm resistor in series between the power pin of the Pi and the case fan lowers the noise, while providing adequate cooling. Wrap the resistor with electrical tape. Make sure the wires are tucked away from the fan.
<br><br>

## Setting up the Raspberry Pi 4
### Option 1: Setup with Monitor (Recommended)
The above YouTube link shows how to setup the Pi 4, however, this assumes you bought from CanaKit. CanaKit pre-installs "Noobs" on the provided MicroSD card; it provides a GUI to install the Raspberry Pi OS without an additional computer. You still require a monitor to setup the Pi. <br><br>
If you rather use your computer to install Raspberry Pi OS directly, or have your own MicroSD card, download the Raspberry Pi Imager from https://www.raspberrypi.com/software/. In the Imager, choose "Raspberry Pi OS (64-bit)". <br>

Once imaged onto the MicroSD card, insert it in the Pi and then connect your mouse, keyboard, and HDMI cable (connect to the port closest to the power port) to continue setup using a monitor and the Pi desktop. Once done, enable SSH under `Preferences > Raspberry Pi Configuration > Interfaces`. <br><br>
**Recommendations**<br>
It’s best to connect the Pi by ethernet as the WiFi card is a little slow. If you do so, I recommend to disable WiFi on the Pi by following the guides on this website https://pimylifeup.com/raspberry-pi-disable-wifi/.

You should also set a static IP address for the Pi in your router settings so it doesn’t change. Having the IP address static is essential to keep all your programs working. <br><br>
**Troubleshooting** <br>
In rare cases, connecting to the monitor won't display anything. If so, try [Setup Headless](#Setup-Headless) or follow this website https://windowsreport.com/raspberry-pi-hdmi-not-working/ by uncommenting the following lines in `boot/config.txt` of the MicroSD card:
* hdmi_force_hotplug=1
* hdmi_drive=2 
<br><br>

### Option 2: Setup Headless
If you want to setup the Pi headless (via SSH terminal instead of a monitor and Pi desktop), in the Raspberry Pi Imager, choose "Raspberry Pi OS Lite (64-bit)". Then in the Advanced options page, enable SSH and Internet.

You could also manually enable SSH and Internet. This website provides a good guide on how to enable SSH and Internet headless https://pimylifeup.com/headless-raspberry-pi-setup/. <br><br>
You can download the required files for SSH and Internet here:
* [ssh](https://github.com/justinknguyen/PiGuide/blob/349dbb43f6d59b7d5426713397d484182c751744/ssh) <br>
* [wpa_supplicant.conf](https://github.com/justinknguyen/PiGuide/blob/349dbb43f6d59b7d5426713397d484182c751744/wpa_supplicant.conf) 
<!-- -->
**Recommendations**<br>
It’s best to connect the Pi by ethernet as the WiFi card is a little slow. If you do so, I recommend to disable WiFi on the Pi by following the guides on this website https://pimylifeup.com/raspberry-pi-disable-wifi/.

You should also set a static IP address for the Pi in your router settings so it doesn’t change. Having the IP address static is essential to keep all your programs working. <br><br>

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