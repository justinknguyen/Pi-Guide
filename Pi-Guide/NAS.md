# NAS

Network-attached Storage (NAS) so you can access a shared storage on your local network from any device.

## Table of Contents

- [Configuration](#configuration)
- [Installation](#installation)
- [Testing](#testing)
- [Sources](#sources)

## Configuration

1. If you have an external drive, you can find it by entering:
   ```
   lsblk
   ```
   - `sda` indicates your external drive, and `mmcblk0` is your micro SD
1. Partition your drive:
   ```
   sudo fdisk /dev/sda
   ```
1. Enter `d` to delete any existing partitions
1. Enter `n` to create a new partition, then enter `p` for primary partition
1. Keep pressing Enter for the next few prompts to choose the default option
1. Enter `Y` to remove the signature if prompted, and then enter `w` to save and exit
1. Format your drive into `ext4` file system:
   ```
   sudo mkfs.ext4 /dev/sda1
   ```
   - If formatting fails, reboot with `sudo reboot` and try again
1. Within the file `sudo nano /etc/fstab` enter the following line at the bottom of the file and save with `Ctrl+X` then `Y`:
   ```
   /dev/sda1 /mnt/sda1/ ext4 defaults,noatime 0 1
   ```
1. Reboot:
   ```
   sudo reboot
   ```
1. Mount your drive:
   ```
   sudo mount /dev/sda1
   ```
   - You can check if it was successfully mounted by entering `lsblk`
   - If it says it can't mount the drive, it could be a false flag. Try rebooting and then check if the drive was mounted with `lsblk`. If it still doesn't say it was mounted, try entering the command again
1. Create a shared folder:
   ```
   sudo mkdir /mnt/sda1/shared
   ```
1. Grant read/write access to shared folder
   ```
   sudo chmod -R 777 /mnt/sda1/shared
   ```

## Installation

1. Install Samba to share your drive over your local network:
   ```
   sudo apt install samba samba-common-bin
   ```
1. Within the file `sudo nano /etc/samba/smb.conf` enter the following lines at the bottom of the file and save with `Ctrl+X` then `Y`:
   ```
   [shared]
   path=/mnt/sda1/shared
   writeable=Yes
   create mask=0777
   directory mask=0777
   public=no
   ```
1. Restart Samba:
   ```
   sudo systemctl restart smbd
   ```
1. Add a Samba password for your user (e.g., my Pi's username is `pi`):
   ```
   sudo smbpasswd -a pi
   ```

## Testing

The following steps uses your Pi's hostname. If it doesn't work, try using the Pi's IP Address instead.

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
  - Select Registered User, then enter your Pi's username and the Samba password you set earlier when prompted55

## Sources

- https://www.raspberrypi.com/tutorials/nas-box-raspberry-pi-tutorial/