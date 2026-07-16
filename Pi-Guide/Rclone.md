# Rclone

Backup anything to any cloud service.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Restoring From Rclone](#restoring-from-rclone)
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
    crontab -e
    ```
    - <ins>IMPORTANT:</ins> use your own crontab here, **not** `sudo crontab -e`. The remote you just configured is saved to your user's config file (`~/.config/rclone/rclone.conf`). A job scheduled under `sudo crontab -e` runs as root instead, which has no such file and can't find `gdrive:` — the backup would silently fail every night even though it worked when you tested it manually above.
1. Add the following line:
    ```
    0 0 * * * rclone copy [FOLDERDIRECTORY] "gdrive:backups" --log-file /home/pi/rclone.log
    ```
    - The `--log-file` flag gives you something to check in [Testing](#testing) below, since cron won't show you any output otherwise.

## Testing

Since the whole point of automating this is that you stop checking it manually, verify the cron job is actually running before trusting it:

1. The morning after your first scheduled run, check the log for errors:
   ```bash
   cat /home/pi/rclone.log
   ```
1. Confirm the files actually landed in your cloud storage:
   ```bash
   rclone ls gdrive:backups
   ```
   Or just check the `backups` folder in Google Drive directly in your browser.

## Restoring From Rclone

Restoring is the same command with the source and destination swapped — `rclone copy` never deletes files, so it's safe to run:

```bash
rclone copy "gdrive:backups" [FOLDERDIRECTORY]
```

To restore to a fresh Pi, first repeat the [Installation](#installation) and [Configuration](#configuration) steps above (installing rclone and re-adding the `gdrive` remote — sign in again if prompted), then run the restore command with your desired destination folder.

If you only want to preview what would change without copying anything yet, add `--dry-run` to either the backup or restore command.

## Sources

- https://gist.githubusercontent.com/sissbruecker/c9263e237f5972e25d9d84b71dd89292/raw/031ef6a4b81cc02f20527a69a46a420ac5afcb16/setup-rclone-gdrive-backup.sh
