# raspiBackup

Automatically backup your Raspberry Pi periodically.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Restoring a Single File](#restoring-a-single-file)
- [Restoring the Whole Pi](#restoring-the-whole-pi)
- [Setup Email Notifications](#setup-email-notifications)
- [Sources](#sources)

## Prerequisites

An external drive to back up to — see [NAS](/Pi-Guide/NAS.md#configuration) for partitioning, formatting, and mounting one (follow the Configuration section up through mounting; you don't need the Samba/Installation part unless you also want network file sharing). This guide assumes it's mounted at `/mnt/sda1`, matching that guide's example.

You can technically skip this and back up to the SD card itself, but that defeats the point — if the card fails, your backups fail with it. Backing up to your Pi's own storage protects you from bad updates or config mistakes, but not from hardware failure.

## Installation

1. Create a backup folder on your mounted external drive:
   ```bash
   sudo mkdir /mnt/sda1/backup
   ```
1. Enter:
   ```bash
   curl -o install -L https://raspibackup.linux-tips-and-tricks.de/install; sudo bash ./install
   ```
1. Press Ok, then select the M2 option to install. Select the I1 option next
1. Press Back and then select the M3 option to configure raspiBackup
1. Select the C2 option and set your backup path:
   ```
   /mnt/sda1/backup
   ```
1. Press Ok and select the C3 option. You can select the "Smart backup strategy" here by pressing space on it. Press Enter twice to go back to the options menu
1. Select the C6 option and configure the services to stop and start. Keeping the default selection is recommended
1. If you want to set up email notifications, see the section below: [Setup Email Notifications](#setup-email-notifications)
1. Select the C9 option, then select R2 to set which day of the week you want to automatically back up. You can adjust the time in the R3 option
1. Once done, go back from the options menu and save the configuration. Finally, press on Finish. You can go back to this screen by entering:
   ```bash
   sudo raspiBackupInstallUI.sh
   ```
1. Run a backup manually:
   ```bash
   sudo raspiBackup -m detailed
   ```

You now have automatic backups running every Sunday at 5 AM to your external drive.

Following this guide as written (without changing the backup type) produces `rsync` backups — the installer's default. This matters for restoring: an rsync backup is just a folder of plain files on your backup drive, which makes it by far the easiest type to restore from (see below). It's also what makes the "Smart backup strategy" from step 6 above fast — only changed files are copied on each run.

## Restoring a Single File

This is the restore you'll actually use most — getting back a config file you broke or accidentally deleted, without touching the SD card. No extra tools needed, since an rsync backup is just a folder tree you can browse.

1. Find your backup folder. Each backup run creates a timestamped folder, for example:
   ```bash
   ls /mnt/sda1/backup/[YOURHOSTNAME]/
   ```
1. Browse the backup like a normal filesystem and copy back what you need. For example, to restore `/etc/pihole/pihole.toml` from a backup made on April 15, 2026:
   ```bash
   sudo cp /mnt/sda1/backup/[YOURHOSTNAME]/[YOURHOSTNAME]-rsync-backup-20260415-050000/etc/pihole/pihole.toml /etc/pihole/pihole.toml
   ```
1. Restart whatever service uses that file (or reboot) to pick up the restored version.

## Restoring the Whole Pi

Use this when the SD card itself has failed or the Pi won't boot. Unlike the single-file restore above, this needs extra hardware and a bit more care.

<ins>What you need:</ins>
- A **new micro SD card** (or other drive) to restore onto — you generally can't restore onto the card that's currently failing/running
- A **USB SD card reader/adapter**, since you'll plug both the backup drive and the new SD card into a running Pi at the same time
- Either this same Pi (booted from a spare SD card with Raspberry Pi OS and raspiBackup freshly installed) or another Pi, since the restore must run from a working Linux system — it can't run from the card it's restoring onto

<ins>IMPORTANT:</ins> the restore command below **erases everything** on the target device once confirmed. Double- and triple-check the device name before pressing Enter — restoring to the wrong device destroys that data instead.

1. Boot a working Pi with raspiBackup installed (see [Installation](#installation) above if it's a fresh SD card).
1. Plug in both the backup drive (containing your backup folders) and the blank SD card you're restoring onto, via a USB reader.
1. Identify the blank SD card's device name — do not guess, confirm it:
   ```bash
   sudo fdisk -l | egrep "^Disk /|^/dev"
   ```
   Match the size shown against your SD card's known size (e.g., `/dev/sda` at 32GB). This is the value you'll use for `-d` below. Your normal boot device (`/dev/mmcblk0`) is never the target.
1. Find your backup drive's mount point and the specific backup folder you want to restore, for example:
   ```bash
   ls /mnt/sdb1/backup/[YOURHOSTNAME]/
   ```
1. Run the restore, replacing `/dev/sda` with your confirmed target device and the path with your chosen backup folder:
   ```bash
   sudo raspiBackup -d /dev/sda /mnt/sdb1/backup/[YOURHOSTNAME]/[YOURHOSTNAME]-rsync-backup-20260415-050000
   ```
   raspiBackup will partition the target device, confirm with you before writing (this is your last chance to abort if the device is wrong), then copy the backup onto it.
1. Once it finishes, move the SD card into the Pi and boot it — it should come up exactly as it was at backup time.

Test this occasionally on a spare SD card before you actually need it — a backup you've never restored from is a backup you don't really know works.

## Setup Email Notifications

1. Install ssmtp and mailutils:
   ```bash
   sudo apt-get install ssmtp
   sudo apt-get install mailutils
   ```
1. Open the file:
   ```bash
   sudo nano /etc/ssmtp/ssmtp.conf
   ```
1. Assuming you have a Gmail account, make sure the file includes the following and save (root and hostname should be already set):
   ```ini
   root=postmaster
   mailhub=smtp.gmail.com:587
   hostname=<your hostname>
   AuthUser=AGmailUserName@gmail.com
   AuthPass=<to be done in the next step if you're using gmail>
   FromLineOverride=YES
   UseSTARTTLS=YES
   ```
1. Go to your Google account settings and then click on the "Security" tab
   - Select the "2-Step Verification" option (enable it if you don't have it already). Then scroll down and select "App passwords"
   - Create an app password. Copy and paste in the app password (without spaces) into `AuthPass` of the `/etc/ssmtp/ssmtp.conf` file from before
1. Send an email to test:
   ```bash
   echo "Hello world email body" | mail -s "Test Subject" AGmailUserName@gmail.com
   ```
1. Re-open the raspiBackup config:
   ```bash
   sudo raspiBackupInstallUI.sh
   ```
1. Configure the email settings:
   - Select M3, then C8, and enter your email address.
   - Select the `ssmtp` option and press Enter.
   - Press Back to save your config, then press Finish.
1. Test it by sending an email after a successful backup:
   ```bash
   sudo raspiBackup -m detailed
   ```

## Sources

- https://www.linux-tips-and-tricks.de/en/installation/
- https://raspberry-projects.com/pi/software_utilities/email/ssmtp-to-send-emails
- https://framps.github.io/raspiBackupDoc/en/restore-intro.html
- https://framps.github.io/raspiBackupDoc/en/how-to-retrieve-single-files-or-directories-from-the-backup.html
- https://framps.github.io/raspiBackupDoc/en/backup-types.html
