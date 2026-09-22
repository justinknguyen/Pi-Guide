# Unattended-Upgrades

Automatically update and upgrade Raspberry Pi.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Update More Than Just Debian](#1-update-more-than-just-debian)
  - [2. Keep the Download Cache from Growing](#2-keep-the-download-cache-from-growing)
- [Pending Reboots](#pending-reboots)
- [Sources](#sources)

## Installation

1. Enter:
   ```bash
   sudo apt-get update
   sudo apt-get install unattended-upgrades
   ```
1. Test with a dry run:
   ```bash
   sudo unattended-upgrade -d -v --dry-run
   ```
1. Enable by selecting `Yes`:
   ```bash
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

## Configuration

Both steps below go in one file of your own, rather than editing the stock `50unattended-upgrades` — that way a package update can't overwrite your changes or prompt you about them.

```bash
sudo nano /etc/apt/apt.conf.d/52unattended-upgrades-local
```

### 1. Update More Than Just Debian

Out of the box, unattended-upgrades only installs updates from Debian itself. Packages from **other** repositories are skipped — which on a Pi includes the kernel and firmware (from the Raspberry Pi repository), and anything you installed from its own repo, like [Docker](/Pi-Guide/Docker.md) or [Tailscale](/Pi-Guide/Tailscale.md).

1. See which repositories your Pi uses:
   ```bash
   apt-cache policy | grep -oE 'o=[^,]+' | sort -u
   ```
1. Add the ones you want updated automatically to your file. For example:
   ```
   Unattended-Upgrade::Origins-Pattern {
       "origin=Raspberry Pi Foundation,codename=${distro_codename}";
       "origin=Tailscale";
       "origin=Docker";
   };
   ```
   These are added to the stock list, not a replacement for it. Only include repos you trust to update without you watching.
1. Confirm they're picked up — the dry run should list the new origins under "Allowed origins":
   ```bash
   sudo unattended-upgrade -d --dry-run 2>&1 | grep -i 'allowed origins'
   ```

### 2. Keep the Download Cache from Growing

Every downloaded package stays in `/var/cache/apt/archives` until something deletes it — on a Pi that's been running a while, this can reach several GB of your SD card. Add to the same file:

```
// Clean up after itself
Unattended-Upgrade::Remove-Unused-Dependencies "true";
APT::Periodic::AutocleanInterval "7";
APT::Periodic::CleanInterval "14";
```

To free the space right away: `sudo apt-get clean`.

## Pending Reboots

Unattended-upgrades installs updates but **does not reboot** by default. Most updates take effect immediately, but kernel and firmware updates only apply after a reboot — until then, they sit installed and doing nothing, possibly for months.

- Check whether a reboot is waiting (the second line lists which packages need it):
  ```bash
  cat /var/run/reboot-required /var/run/reboot-required.pkgs 2>/dev/null
  ```
- **Bootloader firmware is a separate check.** On a Pi 4/5, `rpi-eeprom-update` stages new bootloader firmware to flash on the next boot, and it does *not* create the file above. See if one is staged:
  ```bash
  ls /boot/firmware/pieeprom.upd 2>/dev/null && echo "EEPROM update staged"
  ```
  To cancel a staged update instead of rebooting: `sudo rpi-eeprom-update -r`.
- Rather than remembering to check, have it shown every time you log in — see [Job Monitoring](/Pi-Guide/Job-Monitoring.md#login-status-message).
- Or let it reboot on its own, by adding to your file:
  ```
  Unattended-Upgrade::Automatic-Reboot "true";
  Unattended-Upgrade::Automatic-Reboot-Time "04:00";
  ```
  Only do this if a few minutes of downtime at that hour is fine — e.g., not if this Pi is your household's only [Pi-hole](/Pi-Guide/Pi-hole.md) and people are awake then.

## Sources

- https://www.seancarney.ca/2021/02/06/secure-your-raspberry-pi-by-enabling-automatic-software-updates/
- https://wiki.debian.org/UnattendedUpgrades
- https://github.com/raspberrypi/rpi-eeprom
