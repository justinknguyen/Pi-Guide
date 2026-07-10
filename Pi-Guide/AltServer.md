# AltServer

Run AltServer on your Raspberry Pi to refresh AltStore and sideloaded apps on your iPhone or iPad.

If you have trouble, the tutorial this guide is based on has more detail on parts skipped here: https://gist.github.com/jschiefner/95a22d7f4803e7ad32a95b0f3aa655dc

If you still can't get this to work, try SideStore instead. Otherwise, there's the dockerized version of AltServer for Linux: https://github.com/dreth/Altserver-docker or this simple GUI solution (no WiFi refresh): https://github.com/vyvir/althea

## Table of Contents

- [Prerequisites](#prerequisites)
  - [1. Installing libplist](#1-installing-libplist)
  - [2. Installing libimobiledevice-glue](#2-installing-libimobiledevice-glue)
  - [3. Installing libtatsu](#3-installing-libtatsu)
  - [4. Installing libimobiledevice](#4-installing-libimobiledevice)
  - [5. Installing Rust](#5-installing-rust)
  - [6. Enable Services](#6-enable-services)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Configuring AltServer and AltStore on Your Devices](#1-configuring-altserver-and-altstore-on-your-devices)
  - [2. Configuring Services to Run Automatically on the Pi](#2-configuring-services-to-run-automatically-on-the-pi)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

1. Create a folder on your Pi to store the files that will be downloaded. This guide uses the default path, `/home/pi`:
   ```bash
   mkdir alt-server
   ```
1. cd to your folder. You can double-check if your folder path is `/home/pi/alt-server` by entering `pwd`:
   ```bash
   cd alt-server
   ```
1. Install some things before getting started:
   ```bash
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

1. Install `libplist` from source by first installing dependencies and build tools:
   ```bash
   sudo apt-get install \
       build-essential \
       checkinstall \
       git \
       autoconf \
       automake \
       libtool-bin
   ```
1. Clone and cd into the repository (still inside `alt-server`):
   ```bash
   git clone https://github.com/libimobiledevice/libplist.git
   cd libplist
   ```
1. Build and install it:
   ```bash
   ./autogen.sh
   make
   sudo make install
   ```
1. Go back to the `alt-server` folder:
   ```bash
   cd ..
   ```

### 2. Installing libimobiledevice-glue

1. Install `libimobiledevice-glue` from source, the same way:
   ```bash
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
   ```bash
   git clone https://github.com/libimobiledevice/libimobiledevice-glue.git
   cd libimobiledevice-glue
   ./autogen.sh
   make
   sudo make install
   cd ..
   ```

### 3. Installing libtatsu

1. Install `libtatsu` from source, the same way:
   ```bash
   sudo apt install -y \
       build-essential \
       pkg-config \
       checkinstall \
       git \
       autoconf \
       automake \
       libtool-bin \
       libplist-dev \
       libcurl4-openssl-dev
   export PKG_CONFIG_PATH=/usr/local/lib/pkgconfig
   git clone https://github.com/libimobiledevice/libtatsu
   cd libtatsu
   ./autogen.sh
   make
   sudo make install
   cd ..
   ```

### 4. Installing libimobiledevice

1. Install `libimobiledevice` from source, the same way:
    ```bash
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
    ```bash
    git clone https://github.com/libimobiledevice/libimobiledevice.git
    cd libimobiledevice
    ./autogen.sh
    make
    sudo make install
    cd ..
    ```
1. Run the command to update the links and cache:
    ```bash
    sudo ldconfig
    ```

### 5. Installing Rust

1. Time to install `rustup`. Just enter `1` for the default installation when prompted:
    ```bash
    curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
    ```
1. Reboot your Pi and then cd back to your `alt-server` folder:
    ```bash
    sudo reboot
    ```
    ```bash
    cd alt-server
    ```
1. Setup the toolchain:
    ```bash
    rustup toolchain install stable
    rustup default stable
    ```

### 6. Enable Services

1. Edit the `usbmuxd` service file:
    ```bash
    sudo nano /lib/systemd/system/usbmuxd.service
    ```
1. Enter the following at the end of the service file so it can be enabled with systemctl:
    ```ini
    # Taken from https://bugs.archlinux.org/task/31056
    [Install]
    WantedBy=multi-user.target
    ```
1. Enable the services:
    ```bash
    sudo systemctl enable --now avahi-daemon.service
    sudo systemctl enable --now usbmuxd
    ```

## Installation

1. cd to the `alt-server` folder
   ```bash
   cd alt-server
   ```
1. Install the latest AltServer-Linux `AltServer-aarch64` (for Raspberry Pis) binary from [here](https://github.com/NyaMisty/AltServer-Linux/releases)
   ```bash
   wget https://github.com/NyaMisty/AltServer-Linux/releases/download/v0.0.5/AltServer-aarch64
   ```
1. Install the latest netmuxd (for Raspberry Pis) release from [here](https://github.com/jkcoxson/netmuxd/releases), then extract it:
   ```bash
   wget https://github.com/jkcoxson/netmuxd/releases/download/v0.4.2/netmuxd-aarch64-unknown-linux-gnu.tar.gz
   tar -xzf netmuxd-aarch64-unknown-linux-gnu.tar.gz
   ```
1. While in the `alt-server` folder inside the terminal, enter the following to make the binaries executable:
   ```bash
   chmod +x AltServer-aarch64
   chmod +x netmuxd
   ```
1. Have `Docker` installed on your Pi. If it isn't installed, see the Docker guide linked above. Run the following command to create the Docker container:
   ```bash
   docker run -d --restart always --name anisette-v3 -p 6969:6969 --volume anisette-v3_data:/home/Alcoholic/.config/anisette-v3/lib/ dadoum/anisette-v3-server
   ```

## Configuration

Note: you can do this with multiple devices at a time.

### 1. Configuring AltServer and AltStore on Your Devices

1. Connect your device (iPhone/iPad) to your Mac/PC with a USB cable. In the Finder or in iTunes, enable "Show this iPhone/iPad when on Wi-Fi", then hit `Sync/Apply`.
1. Disconnect your device from USB, and make sure your device is broadcasting itself by checking if your Mac/PC can still see the device in Finder or in iTunes.
   - If you don't see it, you can check if your device is broadcasting itself to your network by entering the following on your Pi:
     ```bash
     sudo apt-get install avahi-utils
     ```
     ```bash
     avahi-browse -a
     ```
   - You can also run the following on Mac. It will show you the "Instance Name" which can be found under "General > About > Wi-Fi Address" of your device:
     ```bash
     dns-sd -B _apple-mobdev2._tcp local
     ```
1. Once your device is broadcasting itself, connect your device to your Pi via USB, and tap `Trust` on the device to trust the Pi. If you are trying to do this on two devices at the same time, only connect one device to the Pi at a time for this step and the next.
1. Verify that the device was paired by entering the following command:
   ```bash
   idevicepair validate
   ```
   - If you get an error: `idevicepair: error while loading shared libraries...`, enter the following again:
     ```bash
     sudo ldconfig
     ```
1. Disconnect your device from the Pi, and run `netmuxd`:
   ```bash
   sudo ./netmuxd
   ```
   - You should see an output like this if it's successful. Ideally, nothing else should be outputting after "Adding device ...", but if it keeps saying "Adding device..." and "Removing device...", you can ignore it, but try restarting the Pi and trying again.
     ```
     Starting netmuxd
     Starting mDNS discovery for _apple-mobdev2._tcp.local with mdns
     Listening on /var/run/usbmuxd
     Adding device 00008110-0123A456B789D012 # or similar, this should appear after a few seconds
     ```
   - If it's not "Adding device", then your device isn't properly broadcasting itself, or isn't properly paired. Retrace your steps and troubleshoot the problem.
1. While `netmuxd` is running, open another terminal tab for your Pi.
1. Download the latest `AltStore.ipa`. As of Apr. 26, 2025, the latest version is 2.2, so replace `2_2` in the below command if there is a more recent version [here](https://faq.altstore.io/release-notes/altstore):
   ```bash
   curl -L https://cdn.altstore.io/file/altstore/apps/altstore/2_2.ipa > AltStore.ipa
   ```
1. Install the ipa file onto your device. Make sure to replace `00008110-0123A456B789D012` with your device's id from Step 5, and include your Apple credentials in the command:
   ```bash
   sudo ALTSERVER_ANISETTE_SERVER=http://127.0.0.1:6969 ./AltServer-aarch64 -u 00008110-0123A456B789D012 -a <apple-id-email> -p <apple-id-password> AltStore.ipa
   ```
   AltStore should now be installed on your device from your Raspberry Pi! The output should be something like this:
   ```
   Installation Progress: 1
   Notify: Installation Succeeded
       AltStore was successfully installed on unknown.
   Finished!
   ```
   If it doesn't finish and freezes, restart the Pi and start from Step 5 again.

### 2. Configuring Services to Run Automatically on the Pi

1. To make `netmuxd` and `AltServer` run on start if the Pi is rebooted, open crontab:
   ```bash
   sudo crontab -e
   ```
1. Then add the following lines. Replace the paths as appropriate to match your environment:
   ```
   @reboot sleep 20 && /home/pi/alt-server/netmuxd > /home/pi/alt-server/log/netmuxd.log 2>&1
   @reboot sleep 20 && ALTSERVER_ANISETTE_SERVER=http://127.0.0.1:6969 /home/pi/alt-server/AltServer-aarch64 > /home/pi/alt-server/log/altserver.log 2>&1
   ```
1. In your `alt-server` folder, make a log folder:
   ```bash
   mkdir log
   ```
1. Do a `sudo reboot` and make sure `netmuxd` and `AltServer` are running with the following lines:
    ```bash
    ps aux | grep netmuxd
    ps aux | grep AltServer
    ```

## Troubleshooting

- Check your `alt-server/log` folder to find and troubleshoot your issue.
- Unfortunately, updating your device may break your setup — it can also just break on its own. If your device stops refreshing:
  1. Check in Finder/iTunes if it can still see your device over WiFi and attempt to sync.
  1. If it can't sync over WiFi, restart your device and clear trusted computers on it.
  1. Redo the "Configuration" steps 1-8, then reboot the Pi.
- The Raspberry Pi appears as `MacbookPro - MacBook Pro 13"` under your Apple ID trusted devices. If you accidentally remove the trusted device, you'll have to start the installation again:
  1. Remove the Docker container and its volume:
     ```bash
     docker stop anisette-v3
     docker rm anisette-v3
     docker volume rm anisette-v3_data
     ```
  1. Reinstall the container:
     ```bash
     docker run -d --restart always --name anisette-v3 -p 6969:6969 --volume anisette-v3_data:/home/Alcoholic/.config/anisette-v3/lib/ dadoum/anisette-v3-server
     ```
  1. Check if the services are running:
     ```bash
     sudo systemctl status avahi-daemon.service
     sudo systemctl status usbmuxd
     ```
  1. If they're not running, enable them:
     ```bash
     sudo systemctl enable --now avahi-daemon.service
     sudo systemctl enable --now usbmuxd
     ```
  1. In your device settings, search for "Clear Trusted Computers" and tap it.
  1. Redo the "Configuration" steps 1-8, then reboot the Pi.

## Sources

- https://gist.github.com/jschiefner/95a22d7f4803e7ad32a95b0f3aa655dc
- https://github.com/libimobiledevice/libplist
- https://github.com/libimobiledevice/libimobiledevice-glue#debian--ubuntu-linux
- https://github.com/libimobiledevice/libimobiledevice#debian--ubuntu-linux
- https://rustup.rs/
- https://github.com/NyaMisty/AltServer-Linux
- https://github.com/jkcoxson/netmuxd
