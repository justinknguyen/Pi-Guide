# Getting Started

## Setting up the Raspberry Pi

Download and install the `Raspberry Pi Imager` from https://www.raspberrypi.com/software/

After installing, insert your micro SD card into your computer and open the Raspberry Pi Imager. Select your Raspberry Pi model and choose your preferred OS. I recommend "Raspberry Pi OS Lite (64-bit)" if you will not use the desktop environment.

After selecting your storage device and clicking on "Next", a popup will appear asking if you want to apply custom settings. In the settings, enter your WiFi credentials and enable SSH under the "Services" tab. You can set your hostname and username/password at this point, or set it later by following the steps at the bottom of this page.

Once the firmware is done flashing to your micro SD, insert it in your Pi and power it on.

## Connecting to the Raspberry Pi

To connect to your Pi and use it, download a terminal to SSH into the Pi, such as `PuTTY` https://www.putty.org/ for Windows users. For Mac users, you can download `Termius` from the app store.

Once you have your app of choice installed, onnect to your Pi by entering it's IP address or hostname; the default hostname is `raspberrypi`. Once connected, you'll be asked to enter your username and password; the default username is `pi` and the password is `raspberry`.

### Change Password

SSH into the Pi and enter the following to change your password: `passwd`

### Change Hostname

1. SSH into Pi, and open the hosts file by entering:
   ```
   sudo nano /etc/hosts
   ```
2. At the bottom, change `raspberrypi` to whatever name you want for the Pi.
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
4. Next, open the hostname file by entering:
   ```
   sudo nano /etc/hostname
   ```
5. Change `raspberrypi` to whatever name you want for the Pi.
6. To save the file, press `Ctrl+X` then `Y` then `Enter`.
7. Reboot:
   ```
   sudo reboot
   ```
