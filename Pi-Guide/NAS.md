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
1. Find your drive's UUID — a permanent ID that doesn't change if you plug in another USB drive (which can shuffle `sda`/`sdb` around):
   ```bash
   sudo blkid /dev/sda1
   ```
   Copy the value after `UUID=` (not `PARTUUID=`).
1. To mount your drive on boot, open the file:
   ```bash
   sudo nano /etc/fstab
   ```
   enter the following line at the bottom of the file, replacing the UUID with yours, and save with `Ctrl+X` then `Y`:
   ```
   UUID=your-uuid-here /mnt/sda1 ext4 defaults,noatime,nofail,x-systemd.device-timeout=10 0 2
   ```
   - `nofail` is the important part on a headless Pi. Without it, if the drive is unplugged or fails, the Pi stops partway through booting and waits for someone at a keyboard — so it never comes back on the network, and you'd have to pull the SD card to fix it.
   - `x-systemd.device-timeout=10` means a missing drive only delays boot by 10 seconds instead of the default 90.
   - The final `2` checks the drive for errors at boot, after the SD card (which is `1`).
1. Check the file for mistakes before rebooting — a broken `fstab` is one of the easiest ways to make a Pi unbootable:
   ```bash
   sudo findmnt --verify
   ```
   It should report `0 parse errors, 0 errors`. A warning that "systemd still uses the old version" is expected — the next step takes care of it. Fix anything else it reports about your new line.
1. Reload the daemon and mount your drive:
   ```bash
   sudo systemctl daemon-reload
   sudo mkdir -p /mnt/sda1
   sudo mount -a
   lsblk
   ```
   - You can check if it was successfully mounted by entering `lsblk`, or `mountpoint /mnt/sda1`
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
1. Add a Samba password for your user (for example, if your Pi's username is `pi`):
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

The same goes for any script that writes to the drive (like a backup): if the drive ever fails to mount, `/mnt/sda1` is just an empty folder on your SD card, and the script will happily fill the SD card instead. Have the script check first — `mountpoint -q /mnt/sda1 || exit 1` — as [Snapshot Backups](/Pi-Guide/Snapshot-Backups.md) does.

## Testing

The following steps use your Pi's hostname. If it doesn't work, try using the Pi's IP address instead. Replace `<hostname>` with your Pi's hostname in each address below.

| Platform | Steps | Address |
| --- | --- | --- |
| macOS | Press `Command+K`, enter the address, then enter your Pi's username and the Samba password you set earlier when prompted | `smb://<hostname>` |
| Windows | Open File Explorer and type the address into the path bar, then enter your Pi's username and the Samba password you set earlier when prompted | `\\<hostname>` |
| iPhone/iPad | Open the Files app, tap the three dots in the top-right, select `Connect to Server`, type the address into the Server field, select "Registered User", then enter your Pi's username and the Samba password you set earlier when prompted | `smb://<hostname>` |

## Sources

- https://www.raspberrypi.com/tutorials/nas-box-raspberry-pi-tutorial/
