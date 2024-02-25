# Rclone

Backup anything to any cloud service.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```
   sudo apt update
   sudo apt upgrade
   ```
2. Install Rclone:
   ```
   sudo apt install rclone
   ```

## Configuration

1. Setup Google Drive as remote:
   ```
   rclone config
   ```
2. Type in `n` for New remote.
3. Type in a name for the remote (e.g., gdrive).
4. Type in `13` for Google Drive.
5. Leave application client id and secret empty.
6. Type in `1` for Full access, or to your preference.
7. Leave the next couple steps empty.
8. Choose default config and enter `N` to auto config.
9. Copy the link given and paste in your web browser. Then paste this verification code back into your terminal.
10. Verify that the configuration is correct and Quit.
11. Backup a folder:
    ```
    rclone copy [FOLDERDIRECTORY] "gdrive:backups"
    ```
12. _Optional:_ Automate backups everyday at midnight:
    ```
    crontab -e
    ```
13. Add the following line:
    ```
    0 0 * * * rclone copy [FOLDERDIRECTORY] "gdrive:backups"
    ```

## Sources

- https://gist.githubusercontent.com/sissbruecker/c9263e237f5972e25d9d84b71dd89292/raw/031ef6a4b81cc02f20527a69a46a420ac5afcb16/setup-rclone-gdrive-backup.sh
