# RetroPie

Turn your Raspberry Pi into a retro-gaming machine!

## Table of Contents

- [Prerequisites](#prerequisites)
- [Configuration](#configuration)
- [Installation](#installation)
- [Testing](#testing)
- [Running ROMs From a USB Drive](#running-roms-from-a-usb-drive)
- [Connecting an Xbox One Controller With Bluetooth](#connecting-an-xbox-one-controller-with-bluetooth)
- [Sources](#sources)

## Prerequisites

The Raspberry Pi will need to be connected to a screen somehow.

- microHDMI to HDMI cable
- Monitor or TV (with built-in speakers or connected speakers)

## Configuration

1. Before starting, update and upgrade your Pi:
   ```bash
   sudo apt update && sudo apt upgrade
   ```
1. Make sure your locale settings are all populated and correct. Check by entering:
   ```bash
   locale
   ```
1. All values should be populated to your correct locale settings. For example, the below indicates "en" for English and "CA" for Canada:
   ```
   LANG=en_CA.UTF-8
   LANGUAGE=en_CA:en
   LC_CTYPE="en_CA.UTF-8"
   LC_NUMERIC="en_CA.UTF-8"
   LC_TIME="en_CA.UTF-8"
   LC_COLLATE="en_CA.UTF-8"
   LC_MONETARY="en_CA.UTF-8"
   LC_MESSAGES="en_CA.UTF-8"
   LC_PAPER="en_CA.UTF-8"
   LC_NAME="en_CA.UTF-8"
   LC_ADDRESS="en_CA.UTF-8"
   LC_TELEPHONE="en_CA.UTF-8"
   LC_MEASUREMENT="en_CA.UTF-8"
   LC_IDENTIFICATION="en_CA.UTF-8"
   LC_ALL=en_CA.UTF-8
   ```
   - if it's not correct, change your locale settings by entering:
     ```bash
     sudo raspi-config
     ```
     then select `5 Localisation Options` and then `L1 Locale`. After selecting your locale, it will ask you to set the default locale for the system environment — choose what you selected.
1. After changing your locale, LANGUAGE and LC_ALL will likely be empty, so set it manually:
   ```bash
   sudo update-locale LANGUAGE=en_CA:en
   sudo update-locale LC_ALL=en_CA.UTF-8
   ```
1. Reboot and check your locale again
   ```bash
   sudo reboot
   ```
   
## Installation

1. Install required packages, download the setup script, then make it executable and run it:
   ```bash
   sudo apt install git lsb-release
   cd
   git clone --depth=1 https://github.com/RetroPie/RetroPie-Setup.git
   cd RetroPie-Setup
   chmod +x retropie_setup.sh
   sudo ./retropie_setup.sh
   ```
1. Once the screen shows, choose the Basic Install. This will likely take 15 minutes or so.
1. (Optional) After it completes, you can install other packages if you want, or you can set it so the RetroPie GUI starts after every boot. You can do so by selecting `C Configuration / tools` and then `autostart` and select the first option to enable it.

Use the following block to re-enter the setup screen if needed:

```bash
cd RetroPie-Setup
sudo ./retropie_setup.sh
```

## Testing

With your Raspberry Pi connected to a screen, start RetroPie by rebooting if you set autostart. Otherwise, you can start it by entering:

```bash
emulationstation
```

## Running ROMs From a USB Drive

This example uses an external ssd connected to a Raspberry Pi. See [NAS](/Pi-Guide/NAS.md) for how to connect and mount an external ssd.

1. Enter:
   ```bash
   lsblk
   ```
1. Check the mountpoint for your USB drive. In this example, it's `/mnt/sda1` for the external ssd.
1. Move the RetroPie folder to your external ssd:
   ```bash
   sudo mv -v /home/pi/RetroPie/* /mnt/sda1/
   ```
1. Check if it was moved:
   ```bash
   ls -l /mnt/sda1
   ```
1. Mount RetroPie to your external ssd by finding the drive's UUID:
   ```bash
   sudo blkid
   ```
   open the fstab file
   ```bash
   sudo nano /etc/fstab
   ```
   and then copy the following in (replace the X's with your UUID)
   ```
   UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXX /home/pi/RetroPie ext4 nofail,defaults 0 0
   ```
1. After saving the file, restart:
   ```bash
   sudo reboot
   ```

## Connecting an Xbox One Controller With Bluetooth

Before beginning, update your Xbox controller with an Xbox or with the "Xbox Accessories" Windows Store app.

1. Run the following command:
   ```bash
   echo 'options bluetooth disable_ertm=Y' | sudo tee -a /etc/modprobe.d/bluetooth.conf
   ```
1. Check if you see `options bluetooth disable_ertm=Y` within the following file:
   ```bash
   sudo nano /etc/modprobe.d/bluetooth.conf
   ```
1. Open the file:
   ```bash
   sudo nano /opt/retropie/configs/all/autostart.sh
   ```
   and add the following line at the TOP of the file (it must be the first line):
   ```bash
   sudo bash -c 'echo 1 > /sys/module/bluetooth/parameters/disable_ertm'
   ```
1. Save and exit by entering `Ctrl+X` then `Y` then `Enter`.
1. Reboot:
   ```bash
   sudo reboot
   ```
1. Connect your Xbox controller by cable, and then run RetroPie by entering:
   ```bash
   emulationstation
   ```
1. After setting up your controller in its GUI, go to the RetroPie settings, then select `Bluetooth`.
1. Select `Remove Bluetooth Device` and remove the Xbox controller if there was an attempt to pair previously
1. Hold the sync button on your Xbox controller to put it into pairing mode, then select `Pair and Connect to Bluetooth Device`, select your controller, then select `DisplayYesNo`.
1. The controller should be paired, and now you'll need to reconfigure this controller again in the RetroPie's start menu.

## Sources

- https://retropie.org.uk/docs/Manual-Installation/
- https://retropie.org.uk/docs/Running-ROMs-from-a-USB-drive/
- https://forums.raspberrypi.com/viewtopic.php?t=213367
