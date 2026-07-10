# Rclone

Backup anything to any cloud service.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```bash
   sudo apt update
   sudo apt upgrade
   ```
1. Install Rclone:
   ```bash
   sudo apt install rclone
   ```

## Configuration

1. Setup Google Drive as remote:
   ```bash
   rclone config
   ```
1. Type in `n` for New remote.
1. Type in a name for the remote (e.g., gdrive).
1. Type in `drive` for Google Drive (more reliable than a menu number, since numbering shifts between versions as new backends are added).
1. Leave application client id and secret empty.
1. Type in `1` for Full access, or to your preference.
1. Leave the next couple steps empty.
1. Choose default config and enter `N` to auto config.
1. Copy the link given and paste in your web browser. Then paste this verification code back into your terminal.
1. Verify that the configuration is correct and Quit.
1. Backup a folder:
    ```bash
    rclone copy [FOLDERDIRECTORY] "gdrive:backups"
    ```
1. (Optional) Automate backups every day at midnight:
    ```bash
    sudo crontab -e
    ```
1. Add the following line:
    ```
    0 0 * * * rclone copy [FOLDERDIRECTORY] "gdrive:backups"
    ```

## Sources

- https://gist.githubusercontent.com/sissbruecker/c9263e237f5972e25d9d84b71dd89292/raw/031ef6a4b81cc02f20527a69a46a420ac5afcb16/setup-rclone-gdrive-backup.sh
