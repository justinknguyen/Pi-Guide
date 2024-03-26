# AltServer

AltServer running on your Raspberry Pi! This is to refresh Altstore and sideloaded apps on your iPhone or iPad.

If you have any trouble, read the tutorial that this guide is based on. It provides more detail on certain parts that I skipped over: https://gist.github.com/jschiefner/95a22d7f4803e7ad32a95b0f3aa655dc

## Table of Contents

- [Prerequisites](#prerequisites)
  - [1. Installing libplist](#1-installing-libplist)
  - [2. Installing libimobiledevice-glue](#2-installing-libimobiledevice-glue)
  - [3. Installing libimobiledevice](#3-installing-libimobiledevice)
  - [4. Installing Rust](#4-installing-rust)
  - [5. Enable Services](#5-enable-services)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Configuring AltServer and AltStore on Your Devices](#1-configuring-altserver-and-altstore-on-your-devices)
  - [2. Configuring Services to Run Automatically on the Pi](#2-configuring-services-to-run-automatically-on-the-pi)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

1. Create a folder on your Pi to store the files we'll be downloading. We'll make one at the default path, which is `/home/pi`:
   ```
   mkdir alt-server
   ```
2. cd to your folder. You can double-check if your folder path is `/home/pi/alt-server` by entering `pwd`:
   ```
   cd alt-server
   ```
3. Install some things before getting started:
   ```
   sudo apt install -y \
     libavahi-compat-libdnssd-dev \
     usbmuxd \
     ninja-build \
     ldc \
     libplist-dev \
     libimobiledevice-dev \
     libgtk-3-0 \
     dub \
     openssl
   ```

### 1. Installing libplist

4. Install `libplist` from source by first installing dependencies and build tools:
   ```
   sudo apt-get install \
       build-essential \
       checkinstall \
       git \
       autoconf \
       automake \
       libtool-bin
   ```
5. Clone and cd to the repository (this is all still inside of your alt-server folder):
   ```
   git clone https://github.com/libimobiledevice/libplist.git
   cd libplist
   ```
6. Build and install it:
   ```
   ./autogen.sh
   make
   sudo make install
   ```
7. You should currently by in the `libplist` folder, so after finishing, go back to the `alt-server` folder:
   ```
   cd ..
   ```

### 2. Installing libimobiledevice-glue

8. Next we need to do a similar process and install `libimobiledevice-glue` from source:
   ```
   sudo apt-get install \
       build-essential \
       pkg-config \
       checkinstall \
       git \
       autoconf \
       automake \
       libtool-bin \
       libplist-dev
   ```
   ```
   git clone https://github.com/libimobiledevice/libimobiledevice-glue.git
   cd libimobiledevice-glue
   ```
   ```
   ./autogen.sh
   make
   sudo make install
   ```
   ```
   cd ..
   ```

### 3. Installing libimobiledevice

9. Next we need to do a similar process again and install `libimobiledevice` from source:
   ```
   sudo apt-get install \
       build-essential \
       pkg-config \
       checkinstall \
       git \
       autoconf \
       automake \
       libtool-bin \
       libplist-dev \
       libusbmuxd-dev \
       libimobiledevice-glue-dev \
       libssl-dev \
       usbmuxd
   ```
   ```
   git clone https://github.com/libimobiledevice/libimobiledevice.git
   cd libimobiledevice
   ```
   ```
   ./autogen.sh
   make
   sudo make install
   ```
   ```
   cd ..
   ```
10. Run the command to update the links and cache:
    ```
    sudo ldconfig
    ```

### 4. Installing Rust

11. Time to install `rustup`. Just enter `1` for the default installation when prompted:
    ```
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```
12. Reboot your Pi and then cd back to your `alt-server` folder:
    ```
    sudo reboot
    ```
    ```
    cd alt-server
    ```
13. Setup the toolchain:
    ```
    rustup toolchain install stable
    rustup default stable
    ```

### 5. Enable Services

14. Edit the `usbmuxd` service file:
    ```
    sudo nano /lib/systemd/system/usbmuxd.service
    ```
15. Enter the following at the end of the service file so it can be enabled with systemctl:
    ```
    # Taken from https://bugs.archlinux.org/task/31056
    [Install]
    WantedBy=multi-user.target
    ```
16. Enable the services:
    ```
    sudo systemctl enable --now avahi-daemon.service
    sudo systemctl enable --now usbmuxd
    ```

## Installation

1. Head to the AltServer-Linux release page [here](https://github.com/NyaMisty/AltServer-Linux/releases) and download the `AltServer-aarch64` binary.
2. Next, head to the netmuxd release page [here](https://github.com/jkcoxson/netmuxd/releases) and download the `aarch64-linux-netmuxd` binary.
3. Place the two binaries within our `alt-server` folder. You can do this easily if you have Netatalk installed or an alternative method of your choice.
4. While in our `alt-server` folder inside the terminal, enter the following to make the binaries executable:
   ```
   chmod +x AltServer-aarch64
   chmod +x aarch64-linux-netmuxd
   ```
5. Have `Docker` installed on your Pi. If you don't have it installed, follow my guide for it. Run the following command to create the Docker container:
   ```
   docker run -d -v lib_cache:/opt/lib/ --restart=always -p 6969:6969 --name anisette dadoum/anisette-server:latest
   ```

## Configuration

Note: you can do this with multiple devices at a time.

### 1. Configuring AltServer and AltStore on Your Devices

1. Connect your device (iPhone/iPad) to your Mac/PC with a USB cable, and enable "Show this iPhone/iPad when on Wi-Fi" in the Finder or in iTunes, then hit `Sync/Apply`.
2. Disconnect your device from USB, and make sure your device is broadcasting itself by checking if your Mac/PC can still see the device in Finder or in iTunes.
   - If you don't see it, you can check if your device is broadcasting itself to your network by entering the following on your Pi:
     ```
     sudo apt-get install avahi-utils
     ```
     ```
     avahi-browse -a
     ```
   - You can also run the following on Mac. It will show you the "Instance Name" which can be found under "General > About > Wi-Fi Address" of your device:
     ```
     dns-sd -B _apple-mobdev2._tcp local
     ```
3. Once your device is broadcasting itself, connect your device to your Pi via USB, and tap `Trust` on the device to trust the Pi. If you are trying to do this on two devices at the same time, just do one at a time (only have one device connected to the Pi at a time) for this step and the next step.
4. Verify that the device was paired by entering the following command:
   ```
   idevicepair validate
   ```
   - If you get an error: `idevicepair: error while loading shared libraries...`, enter the following again:
     ```
     sudo ldconfig
     ```
5. Disconnect your device from the Pi, and run `netmuxd`:
   ```
   sudo ./aarch64-linux-netmuxd
   ```
   - You should see an output like this if it's successful. Ideally, nothing else should be outputing after "Adding device ...", but if it keeps saying "Adding device..." and "Removing device...", you can ignore it but try restarting the Pi and try again.
     ```
     Starting netmuxd
     Starting mDNS discovery for _apple-mobdev2._tcp.local with mdns
     Listening on /var/run/usbmuxd
     Adding device 00008110-0123A456B789D012 # or similar, this should appear after a few seconds
     ```
   - If it's not "Adding device", then your device isn't properly broadcasting itself or paired. Retrace your steps and troubleshoot the problem.
6. While `netmuxd` is running, open another terminal tab for your Pi, and install the `AltStore` app to your devices (note: replace `00008110-0123A456B789D012` with your device code from the above step):
   ```
   curl -L https://cdn.altstore.io/file/altstore/apps/altstore/1_6_3.ipa > AltStore.ipa
   ```
   - Replace 1_6_3 with the most recent version
   - You can find the latest AltStore version here https://faq.altstore.io/release-notes/altstore
     ```
     sudo ALTSERVER_ANISETTE_SERVER=http://127.0.0.1:6969 ./AltServer-aarch64 -u 00008110-0123A456B789D012 -a <apple-id-email> -p <apple-id-password> AltStore.ipa
     ```
   - Replace `00008110-0123A456B789D012` with your device code from the above step
   - If your Apple ID password ends with !! or !?, the command might not run. Change the ending of your password to letters.
   - For the first time, you'll likely need to enter a 2FA code. Enter it in the terminal when prompted. After verifying, the Raspberry Pi appears as `MacbookPro - MacBook Pro 13"` under your Apple ID trusted devices.
   - AltStore should now be installed on your device from your Raspberry Pi! The output should be something like this:
     ```
     Installation Progress: 1
     Notify: Installation Succeeded
         AltStore was successfully installed on unknown.
     Finished!
     ```
   - If it doesn't finish and freezes, restart the Pi and start from Step 5 again.

### 2. Configuring Services to Run Automatically on the Pi

7. We need to make `netmuxd` and `AltServer` run on start if the Pi is rebooted. Open up crontab:
   ```
   sudo crontab -e
   ```
8. Then add the following lines:
   ```
   @reboot sleep 20 && /home/pi/alt-server/aarch64-linux-netmuxd > /home/pi/alt-server/log/netmuxd.log 2>&1
   @reboot sleep 20 && ALTSERVER_ANISETTE_SERVER=http://127.0.0.1:6969 /home/pi/alt-server/AltServer-aarch64 > /home/pi/alt-server/log/altserver.log 2>&1
   ```
9. In your `alt-server` folder, make a log folder:
   ```
   mkdir log
   ```
10. Do a `sudo reboot` and make sure `netmuxd` and `AltServer` are running with the following lines:
    ```
    ps aux | grep netmuxd
    ps aux | grep AltServer
    ```

## Troubleshooting

- The Raspberry Pi appears as `MacbookPro - MacBook Pro 13"` under your Apple ID trusted devices. If you accidentally remove the trusted device, you'll have to start the installation again by first removing the docker container and reinstalling the anisette server. Here's how you can remove it:
  ```
  docker stop anisette
  docker rm anisette
  docker volume rm lib_cache
  ```
  Then reinstall the container:
  ```
  docker run -d -v lib_cache:/opt/lib/ --restart=always -p 6969:6969 --name anisette dadoum/anisette-server:latest
  ```
  Check if the services are running:
  ```
  sudo systemctl status avahi-daemon.service
  sudo systemctl status usbmuxd
  ```
  If they're not running, enter the following:
  ```
  sudo systemctl enable --now avahi-daemon.service
  sudo systemctl enable --now usbmuxd
  ```
  Next, search in your device settings for "Clear Trusted Computers" and tap it to clear it. Finally, do the "Configuration" steps 1-6 again and reboot the Pi.
- Unfortunately, updating your device may break your setup. It could also just break on its own. If your device stops refreshing, check in Finder/iTunes if it can still see your device over WiFi and attempt to sync. If it can't sync over WiFi, you'll have to restart your device and clear trusted computers on it, then do "Configuration" steps 1-6 again and reboot the Pi.

## Sources

- https://gist.github.com/jschiefner/95a22d7f4803e7ad32a95b0f3aa655dc
- https://github.com/libimobiledevice/libplist
- https://github.com/libimobiledevice/libimobiledevice-glue#debian--ubuntu-linux
- https://github.com/libimobiledevice/libimobiledevice#debian--ubuntu-linux
- https://rustup.rs/
- https://github.com/NyaMisty/AltServer-Linux
- https://github.com/jkcoxson/netmuxd
