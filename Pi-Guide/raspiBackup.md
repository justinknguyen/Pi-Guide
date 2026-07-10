# raspiBackup

Automatically backup your Raspberry Pi periodically.

## Table of Contents

- [Installation](#installation)
- [Setup Email Notifications](#setup-email-notifications)
- [Sources](#sources)

## Installation

1. First create a backup folder. For example, this guide uses an external ssd mounted on `/mnt/sda1`:
   ```bash
   sudo mkdir /mnt/sda1/backup
   ```
1. Enter:
   ```bash
   curl -o install -L https://raspibackup.linux-tips-and-tricks.de/install; sudo bash ./install
   ```
1. Press Ok and then select the M2 option to install, and then select the I1 option
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
   - Select the "2-Step Verification" option (enable it if you don't have it already), and then scroll down and select "App passwords"
   - Create an app password. Copy and paste in the app password (without spaces) into `AuthPass` of the `/etc/ssmtp/ssmtp.conf` file from before
1. Send an email to test:
   ```bash
   echo "Hello world email body" | mail -s "Test Subject" AGmailUserName@gmail.com
   ```
1. Re-open the raspiBackup config:
   ```bash
   sudo raspiBackupInstallUI.sh
   ```
1. Select M3, then C8 and enter your email address that you set up. Select the `ssmtp` option and press Enter, then press Back and save your config. Finally, press Finish
1. Test it by sending an email after a successful backup:
   ```bash
   sudo raspiBackup -m detailed
   ```

## Sources

- https://www.linux-tips-and-tricks.de/en/installation/
- https://raspberry-projects.com/pi/software_utilities/email/ssmtp-to-send-emails
