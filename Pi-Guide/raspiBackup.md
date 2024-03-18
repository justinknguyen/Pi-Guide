# raspiBackup

Automatically backup your Raspberry Pi periodically.

## Table of Contents

- [Installation](#installation)
- [Setup Email Notifications](#setup-email-notifications)
- [Sources](#sources)

## Installation

1. First create a backup folder. For example, I have an external ssd that I have mounted on `/mnt/sda1`:
   ```
   sudo mkdir /mnt/sda1/backup
   ```
1. Enter:
   ```
   curl -o install -L https://raspibackup.linux-tips-and-tricks.de/install; sudo bash ./install
   ```
1. Press Ok and then select the M2 option to install, and then select the I1 option
1. Press Back and the select the M3 option to configure raspiBackup
1. Select the C2 option and set your backup path:
   ```
   /mnt/sda1/backup
   ```
1. Press Ok and select the C3 option. You can select the "Smart backup strategy" here by pressing space on it. Press Enter twice to go back to the options menu
1. Select the C6 option and configure the services to stop and start. I would recommend to keep the default selection
1. If you want to setup email notifications, see the following section below [Setup Email Notifications](#setup-email-notifications)
1. Select the C9 option, then select R2 to set which day of the week you want to automatically backup. You can adjust the time in the R3 option
1. Once done, go back from the options menu and save the configuration. Finally, press on Finish. You can go back to this screen by entering:
   ```
   sudo raspiBackupInstallUI.sh
   ```
1. Run a backup manually:
   ```
   sudo raspiBackup -m detailed
   ```

Congrats! If you followed everything, you should have automatic backups setup that will run every Sunday at 5 AM to your external drive.

## Setup Email Notifications

1. Install ssmtp and mailutils:
   ```
   sudo apt-get install ssmtp
   sudo apt-get install mailutils
   ```
1. Open the file:
   ```
   sudo nano /etc/ssmtp/ssmtp.conf
   ```
1. Assuming you have a gmail, make sure the file includes the following and save (root and hostname should be already set):
   ```
   root=postmaster
   mailhub=smtp.gmail.com:587
   hostname=<your hostname>
   AuthUser=AGmailUserName@gmail.com
   AuthPass=<to be done in the next step if you're using gmail>
   FromLineOverride=YES
   UseSTARTTLS=YES
   ```
1. Go to your google account settings and then click on the "Security" tab
   - Select the "2-Step Verification" option (enable it if you don't have it already), and then scroll down and select "App passwords"
   - Create an app password. Copy and paste in the app password (without spaces) into `AuthPass` of the `/etc/ssmtp/ssmtp.conf` file from before
1. Send an email to test:
   ```
   echo "Hello world email body" | mail -s "Test Subject" AGmailUserName@gmail.com
   ```
1. Re-open the raspiBackup config:
   ```
   sudo raspiBackupInstallUI.sh
   ```
1. Select M3, then C8 and enter your email address that you setup. Select the `ssmtp` option and press Enter, then press Back and save your config. Finally, press Finish
1. Test it sending an email after a successful backup:
   ```
   sudo raspiBackup -m detailed
   ```

## Sources

- https://www.linux-tips-and-tricks.de/en/installation/
- https://raspberry-projects.com/pi/software_utilities/email/ssmtp-to-send-emails
