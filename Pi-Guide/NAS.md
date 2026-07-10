# NAS

Network-attached Storage (NAS) so you can access shared storage on your local network from any device.

## Table of Contents

- [Configuration](#configuration)
- [Installation](#installation)
- [Docker Containers Depending on External Drive](#docker-containers-depending-on-external-drive)
- [Testing](#testing)
- [Sources](#sources)

## Configuration

If you have an external ssd, your Pi may have trouble booting due to static on the USB 3.0 port. Either plug it into a USB 2.0 port or put a USB hub between the ssd and the Pi. Fast modern external ssds need a powered USB hub, since the Pi doesn't supply enough power on its own.

1. If you have an external drive, you can find it by entering:
   ```bash
   lsblk
   ```
   - `sda` indicates your external drive, and `mmcblk0` is your micro SD
1. Partition your drive:
   ```bash
   sudo fdisk /dev/sda
   ```
1. Enter `d` to delete any existing partitions
1. Enter `n` to create a new partition, then enter `p` for primary partition
1. Keep pressing Enter for the next few prompts to choose the default option
1. Enter `Y` to remove the signature if prompted, and then enter `w` to save and exit
1. Format your drive into `ext4` file system:
   ```bash
   sudo mkfs.ext4 /dev/sda1
   ```
   - If formatting fails, reboot with `sudo reboot` and try again
1. To mount your drive on boot, open the file:
   ```bash
   sudo nano /etc/fstab
   ```
   enter the following line at the bottom of the file and save with `Ctrl+X` then `Y`:
   ```
   /dev/sda1 /mnt/sda1 ext4 defaults,noatime 0 1
   ```
   - If your Pi is not mounting on boot, enter the following command to obtain its UUID:
     ```bash
     sudo blkid
     ```
     and then enter the following within the `fstab` file instead:
     ```
     UUID=your-uuid-here /mnt/sda1 ext4 defaults,auto,users,rw,nofail 0 0
     ```
1. Reload the daemon and mount your drive:
   ```bash
   sudo systemctl daemon-reload
   sudo mkdir -p /mnt/sda1
   sudo mount /dev/sda1 /mnt/sda1
   lsblk
   ```
   - Use the following to mount if you used the UUID instead:
     ```bash
     sudo mount -a
     ```
   - You can check if it was successfully mounted by entering `lsblk`
   - If it says it can't mount the drive, it could be a false alarm. Try rebooting and then check if the drive was mounted with `lsblk`. If it still doesn't say it was mounted, try entering the command again
1. Create a shared folder and grant it read/write access:
   ```bash
   sudo mkdir /mnt/sda1/shared
   sudo chmod -R 777 /mnt/sda1/shared
   ```

## Installation

1. Install Samba to share your drive over your local network:
   ```bash
   sudo apt install samba samba-common-bin
   ```
1. Open the file
   ```bash
   sudo nano /etc/samba/smb.conf
   ```
   enter the following lines at the bottom of the file and save with `Ctrl+X` then `Y`:
   ```ini
   [shared]
   path=/mnt/sda1/shared
   writeable=Yes
   create mask=0777
   directory mask=0777
   public=no
   ```
1. Restart Samba:
   ```bash
   sudo systemctl restart smbd
   ```
1. Add a Samba password for your user (e.g., my Pi's username is `pi`):
   ```bash
   sudo smbpasswd -a pi
   ```

## Docker Containers Depending on External Drive

If you have Docker containers that depend on your external drive, you will need to delay the startup of your Docker service until your Pi mounts that external drive.

1. Create an override file:
   ```bash
   sudo systemctl edit docker.service
   ```
1. After the first two commented lines, enter the following:
   ```ini
   [Unit]
   RequiresMountsFor=/mnt/sda1
   ```
   - replace the path accordingly

## Testing

The following steps use your Pi's hostname. If it doesn't work, try using the Pi's IP Address instead.

- To access your NAS from macOS, press `Command+K` and type in the following (replace `<hostname>` with your Pi's hostname)
  ```
  smb://<hostname>
  ```
  - Then enter your Pi's username and the Samba password you set earlier when prompted
- To access your NAS from Windows, open File explorer and type in the following in the path bar (replace `<hostname>` with your Pi's hostname)
  ```
  \\<hostname>
  ```
  - Then enter your Pi's username and the Samba password you set earlier when prompted
- To access your NAS from your iPhone or iPad, open the Files app and press on the three dots in the top-right, then select `Connect to Server`. Type in the following for the Server field (replace `<hostname>` with your Pi's hostname)
  ```
  smb://<hostname>
  ```
  - Select Registered User, then enter your Pi's username and the Samba password you set earlier when prompted

## Sources

- https://www.raspberrypi.com/tutorials/nas-box-raspberry-pi-tutorial/
