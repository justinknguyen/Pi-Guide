# Snapshot Backups

A lightweight, file-level backup with a script you can read end to end. It keeps **daily snapshots** on a USB drive — so you can go back to any recent day — plus an optional **mirror on a NAS** as a copy that survives the Pi itself dying.

Compared to [raspiBackup](/Pi-Guide/raspiBackup.md): raspiBackup images the whole system and can restore a bootable SD card in one step. This backs up only the folders that matter (configs, Docker data, your home folder) plus lists of what's installed, and rebuilds onto a fresh OS. It's smaller, faster, easy to browse, and doesn't stop any services while it runs.

## Table of Contents

- [Prerequisites](#prerequisites)
- [How It Works](#how-it-works)
- [Installation](#installation)
- [Adding a NAS Copy](#adding-a-nas-copy)
- [Testing](#testing)
- [Restoring](#restoring)
- [Sources](#sources)

## Prerequisites

- An external drive mounted on the Pi — see [NAS](/Pi-Guide/NAS.md#configuration) for formatting and mounting one (this guide assumes `/mnt/sda1`). It must be formatted `ext4`: snapshots rely on hardlinks and Linux file permissions, which FAT/exFAT drives don't support.
- Optional: a NAS or another computer to copy to.
- Recommended: [Job Monitoring](/Pi-Guide/Job-Monitoring.md), so you hear about it when a backup fails.

## How It Works

The two destinations do different jobs, and you want both:

- **USB = snapshots.** Each day gets its own folder, `snapshots/2026-09-22/`, with `latest` pointing at the newest. If you delete or break a file and don't notice for a week, last week's snapshot still has the good copy. A plain mirror can't do that — it copies the deletion or the corrupted file on its next run.
- **NAS = mirror.** One copy of today, overwritten each run. It's there for when the Pi and its USB drive are both gone (failure, theft, fire).

Snapshots are cheap because of `rsync --link-dest`: a file that hasn't changed since yesterday isn't copied again, it's *hardlinked* — the new snapshot points at the same data on disk. Thirty snapshots of mostly unchanged files take up barely more space than one. Each snapshot is still a complete, independent folder; deleting an old one never damages the others.

The script keeps the 14 most recent snapshots, then one per week for 8 more weeks, and deletes the rest.

## Installation

1. Create the script:
   ```bash
   sudo nano /usr/local/sbin/snapshot-backup.sh
   ```
1. Paste the following in, then edit the settings at the top — especially `SOURCES`, the list of folders to back up:
   ```bash
   #!/bin/bash
   # Backup to two places:
   #   USB drive : dated, hardlinked snapshots -> go back to any recent day
   #   NAS       : plain mirror of today       -> a copy off this Pi
   set -uo pipefail

   # ---- settings ---------------------------------------------------------------
   USB_MOUNT="/mnt/sda1"                          # the drive's mount point
   USB_ROOT="$USB_MOUNT/backups/$(hostname)"      # where snapshots go on it
   NAS_HOST=""                                    # NAS address; empty = no NAS copy
   NAS_TARGET=""                                  # where on the NAS (examples below)
   NAS_PASS=""                                    # rsync daemon password file, if any
   STATE_DIR="/var/backups/system-state"
   CONF="/etc/job-monitoring.conf"                # optional, see Job Monitoring guide
   LOCK="/run/lock/snapshot-backup.lock"

   KEEP_RECENT=14   # keep this many most-recent daily snapshots...
   KEEP_WEEKLY=8    # ...then one per week for this many weeks

   # What to back up. Each line: <label>|<folder>|<comma-separated excludes>
   SOURCES=(
     "home|/home/pi/|.cache,.bash_history"
     "etc|/etc/|pihole/pihole-FTL.db*"
     "crontabs|/var/spool/cron/crontabs/|"
     "docker-volumes|/var/lib/docker/volumes/|backingFsBlockDev"
     "local-sbin|/usr/local/sbin/|"
   )
   # -----------------------------------------------------------------------------

   RSYNC_OPTS=(-a --delete --numeric-ids)
   SNAP_ROOT="$USB_ROOT/snapshots"
   TODAY="$(date +%F)"   # fixed once, so a run past midnight stays in one snapshot
   rc=0

   log()  { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }
   warn() { log "WARN: $*"; rc=1; }

   # rsync exit 24 means "some files vanished before they could be copied". That is
   # normal when backing up running services (a log or temp file was rotated away
   # mid-run), and the copy is otherwise complete, so it counts as success.
   copy() {
       rsync "$@"; local r=$?
       (( r == 24 )) && { log "note: some files vanished during the copy (rsync 24)"; r=0; }
       return "$r"
   }

   log "start"
   trap 'log "exit=$?"' EXIT
   trap 'exit 129' HUP; trap 'exit 130' INT; trap 'exit 143' TERM

   [[ -r $CONF ]] && . "$CONF"
   hc() {
       local url="${HC_BACKUP_URL:-}"
       [[ -z $url ]] && return 0
       curl -fsS -m 10 --retry 3 -o /dev/null "${url}${1:-}" 2>/dev/null || true
   }

   # One run at a time. If yesterday's run is stuck, it still holds the lock
   # (rsync inherits it), so today's run fails loudly instead of piling on.
   exec 9>"$LOCK"
   if ! flock -n 9; then
       log "WARN: previous run still holds $LOCK - skipping this run"
       hc /fail
       exit 1
   fi

   # Lists of what's installed and enabled -- enough to rebuild on a fresh card.
   # A failed capture leaves an empty or stale list, which is useless for a
   # rebuild, so it counts as a warning rather than being silently ignored.
   capture_state() {
       mkdir -p "$STATE_DIR" || { warn "cannot create $STATE_DIR"; return; }
       cap() {
           local file=$1; shift
           "$@" > "$STATE_DIR/$file" 2>/dev/null || warn "system-state: $file failed ($1)"
       }
       cap packages-manual.list  apt-mark showmanual
       cap packages-all.list     dpkg --get-selections
       cap services-enabled.list systemctl list-unit-files --state=enabled --no-legend
       command -v docker >/dev/null &&
           cap docker-containers.list docker ps -a --format '{{.Names}}\t{{.Image}}\t{{.Ports}}'
       cap blkid.txt             blkid
       cap fstab                 cat /etc/fstab
   }

   excludes() {
       local IFS=',' p
       for p in $1; do [[ -n $p ]] && printf -- '--exclude=%s\n' "$p"; done
   }

   # Newest snapshot before today, to hardlink unchanged files against.
   previous_snapshot() {
       find "$SNAP_ROOT" -mindepth 1 -maxdepth 1 -type d -name '20??-??-??' 2>/dev/null \
           | grep -v "/$TODAY\$" | sort | tail -1
   }

   prune_snapshots() {
       local all keep=() seen=() d week n=0
       mapfile -t all < <(find "$SNAP_ROOT" -mindepth 1 -maxdepth 1 -type d -name '20??-??-??' | sort -r)
       keep=("${all[@]:0:$KEEP_RECENT}")
       for d in "${all[@]:$KEEP_RECENT}"; do
           week=$(date -d "$(basename "$d")" +%G-W%V 2>/dev/null) || continue
           [[ " ${seen[*]-} " == *" $week "* ]] && continue
           seen+=("$week"); keep+=("$d"); n=$((n+1))
           (( n >= KEEP_WEEKLY )) && break
       done
       for d in "${all[@]}"; do
           [[ " ${keep[*]-} " == *" $d "* ]] && continue
           # Safety: only ever delete a dated folder directly inside snapshots/.
           [[ $d == "$SNAP_ROOT"/20??-??-?? ]] || { warn "refusing to prune $d"; continue; }
           # A failed prune is a warning: unnoticed, snapshots pile up until the drive fills.
           if rm -rf -- "$d"; then log "pruned $(basename "$d")"; else warn "could not prune $d"; fi
       done
   }

   hc /start
   capture_state
   SOURCES+=("system-state|$STATE_DIR/|")

   # ---- USB: dated snapshots ---------------------------------------------------
   # If the drive isn't mounted, $USB_MOUNT is just an empty folder on the SD
   # card -- writing there would silently fill the SD card. Never skip this check.
   if mountpoint -q "$USB_MOUNT"; then
       mkdir -p "$SNAP_ROOT"
       PREV="$(previous_snapshot)"
       usb_ok=1
       for entry in "${SOURCES[@]}"; do
           IFS='|' read -r label src excl <<< "$entry"
           [[ -e $src ]] || { warn "$label: $src missing, skipped"; usb_ok=0; continue; }
           dest="$SNAP_ROOT/$TODAY/$label"
           mkdir -p "$dest"
           opts=("${RSYNC_OPTS[@]}")
           mapfile -t -O "${#opts[@]}" opts < <(excludes "$excl")
           [[ -n $PREV && -d $PREV/$label ]] && opts+=(--link-dest="$PREV/$label")
           if copy "${opts[@]}" "$src" "$dest/"; then
               log "USB $label OK"
           else
               warn "USB $label FAILED"; usb_ok=0
           fi
       done
       # Only point "latest" at a complete snapshot. A partial one is kept under
       # its date, but "latest" stays on the last good one.
       if (( usb_ok )); then
           ln -sfn "snapshots/$TODAY" "$USB_ROOT/latest"
       else
           warn "snapshot $TODAY incomplete - latest left at $(readlink "$USB_ROOT/latest")"
       fi
       prune_snapshots
   else
       warn "$USB_MOUNT is not mounted - skipped USB snapshots"
   fi

   # ---- NAS: mirror of today ---------------------------------------------------
   # Checked first so that when the NAS is down, the USB backup above still
   # counts and the NAS is just reported as skipped.
   if [[ -z $NAS_HOST ]]; then
       :
   elif ! ping -c1 -W3 "$NAS_HOST" >/dev/null 2>&1; then
       warn "NAS $NAS_HOST unreachable - skipped NAS copy"
   else
       for entry in "${SOURCES[@]}"; do
           IFS='|' read -r label src excl <<< "$entry"
           [[ -e $src ]] || continue
           # Give up if the NAS stops responding, instead of hanging forever.
           opts=("${RSYNC_OPTS[@]}" --timeout=300)
           [[ $NAS_TARGET == *::* ]] && opts+=(--contimeout=30)   # rsync daemon only
           [[ -n $NAS_PASS ]] && opts+=(--password-file="$NAS_PASS")
           mapfile -t -O "${#opts[@]}" opts < <(excludes "$excl")
           copy "${opts[@]}" "$src" "$NAS_TARGET/$label/" && log "NAS $label OK" || warn "NAS $label FAILED"
       done
   fi

   if (( rc == 0 )); then hc; else hc /fail; fi
   exit $rc
   ```
   Notes on the settings:
   - The example `SOURCES` cover the usual things worth keeping: your home folder, all of `/etc` (system and service configs, including [Pi-hole](/Pi-Guide/Pi-hole.md)'s), your crontabs, [Docker](/Pi-Guide/Docker.md) volumes, and your own scripts. Add a line for anything else, such as a folder of compose files or a service's data folder.
   - Excludes are relative to that source folder. Leave out things that are big and can be regenerated — caches, and Pi-hole's query log database (`pihole-FTL.db`, often hundreds of MB).
   - `.bash_history` is excluded because anything ever typed on the command line — including a password — ends up in it. The same goes for any other file that collects secrets by accident.
   - The script also saves lists of your installed packages, enabled services, containers and disks into `system-state/`, which is what makes a rebuild on a fresh SD card practical.
1. Save it and make it root-only. It runs as root so it can read everything in `/etc`:
   ```bash
   sudo chmod 700 /usr/local/sbin/snapshot-backup.sh
   ```
1. Run it once by hand (the first run copies everything, so it takes the longest):
   ```bash
   sudo /usr/local/sbin/snapshot-backup.sh
   ```
1. Schedule it daily — in **root's** crontab — with its output going to a log:
   ```bash
   sudo crontab -e
   ```
   ```
   30 4 * * * /usr/local/sbin/snapshot-backup.sh >> /var/log/snapshot-backup.log 2>&1
   ```
   Then set up log rotation and monitoring for it as described in [Job Monitoring](/Pi-Guide/Job-Monitoring.md). The script already includes the healthchecks.io pings — it reads `HC_BACKUP_URL` from `/etc/job-monitoring.conf` if that file exists.

A few safety checks in the script are there for a reason. Don't remove them:

- **`mountpoint -q`**: if the USB drive ever fails to mount, `/mnt/sda1` is just an empty folder on the SD card. Without this check, the backup would quietly copy everything onto your SD card until it filled up.
- **The pruning check** only ever deletes a folder named like a date directly inside `snapshots/`. A typo in a path setting can't turn it into `rm -rf` on something else.
- **The `flock` lock** keeps two runs from overlapping. If a run gets stuck, the next day's run logs a warning and reports a failure instead of starting a second copy alongside it.
- **`latest` only moves on a complete run.** If any source fails, that day's folder is kept but `latest`, which is what you'd restore from, still points at the last complete snapshot.

## Adding a NAS Copy

Set `NAS_HOST` to the NAS's IP address and `NAS_TARGET` to where the copy should go. Pick whichever your NAS supports:

- **Over SSH** (works with most NASes and any Linux machine):
  ```bash
  NAS_HOST="192.168.50.20"
  NAS_TARGET="backupuser@192.168.50.20:/volume1/backups/$(hostname)"
  ```
  The script runs as root, so **root** needs an SSH key the NAS accepts: run `sudo ssh-keygen -t ed25519` (press Enter for no passphrase), then `sudo ssh-copy-id backupuser@192.168.50.20`, then test with `sudo ssh backupuser@192.168.50.20` once to accept its host key.
- **rsync daemon** (e.g. a Synology/QNAP "rsync server", or a `media` module you set up yourself):
  ```bash
  NAS_HOST="192.168.50.20"
  NAS_TARGET="backupuser@192.168.50.20::backups/$(hostname)"
  NAS_PASS="/etc/snapshot-backup-rsync.pass"
  ```
  Put the rsync password alone in that file, and `sudo chmod 600` it — rsync refuses a password file other users can read.

A few things to know:

- The NAS reachability check means a NAS that's off or rebooting doesn't stop the USB backup; it's logged as a warning and the run is reported as failed so you notice.
- `--timeout=300` ends the NAS copy if the NAS stops responding for 5 minutes. Without it, a NAS that goes away mid-copy can leave rsync waiting forever. `--contimeout` does the same for the initial connection, but rsync only accepts it for rsync-daemon (`::`) targets and errors out over SSH, so the script adds it only for those.
- `--numeric-ids` keeps file owners correct. Without it, rsync matches owners **by name**, and if the NAS has its own user called `pi` (or anything else) with a different ID number, restored files come back owned by the wrong user. Keep it.
- **Your backup contains secrets**: password hashes (`/etc/shadow`), the Pi's SSH host keys, API tokens and passwords in service configs. File permissions are preserved, but anyone with admin access to the NAS — or any machine that mounts that share — can read them. Give the backup its own share or folder that only you can access. If that isn't enough, encrypt the backup (e.g. use [BorgBackup](https://www.borgbackup.org/) or restic instead of plain rsync).
- rsync's `--delete` doesn't touch excluded paths. If you remove a source from the list or add a new exclude, the old copy stays in the NAS mirror until you delete it by hand.

## Testing

A backup you've never restored from is a backup you don't know works. Once after setup, and again now and then:

1. Check the log ends with `exit=0` and every source says OK:
   ```bash
   sudo tail -20 /var/log/snapshot-backup.log
   ```
1. Check snapshots are sharing space. `du` counts each hardlinked file once, so the total should be only a little more than one snapshot:
   ```bash
   sudo du -sh /mnt/sda1/backups/$(hostname)/snapshots/*
   ```
   (The first folder listed shows the full size, the others show only what changed.)
1. Restore the latest snapshot to a scratch folder, and check that permissions survived and files match what's live:
   ```bash
   sudo rsync -a /mnt/sda1/backups/$(hostname)/latest/ /tmp/restore-test/
   sudo ls -l /tmp/restore-test/etc/shadow                # should be owned by root:shadow, not you
   sudo md5sum /etc/fstab /tmp/restore-test/etc/fstab     # the two hashes should match
   ```
1. If you back up any SQLite databases (e.g. Pi-hole's `gravity.db`), make sure the restored copy isn't corrupt:
   ```bash
   sudo pihole-FTL sqlite3 /tmp/restore-test/etc/pihole/gravity.db "PRAGMA integrity_check;"
   ```
   It should print `ok`. Note that copying a database while its service is writing to it can occasionally capture it mid-write — this check is how you'd find out.
1. Clean up: `sudo rm -rf /tmp/restore-test`

## Restoring

Everything is plain files, so there are no special tools. Snapshots are at `/mnt/sda1/backups/[YOURHOSTNAME]/snapshots/[DATE]/`, and `latest` is the newest.

**A single file or folder** — copy it back, then restart whatever service uses it:

```bash
sudo cp -a /mnt/sda1/backups/$(hostname)/snapshots/2026-09-15/etc/pihole/pihole.toml /etc/pihole/pihole.toml
```

To see how a file has changed over time, compare two snapshots: `diff snapshots/2026-09-15/etc/fstab snapshots/2026-09-22/etc/fstab`.

**The whole Pi, onto a fresh SD card** — this rebuilds rather than clones, so do it step by step:

1. Flash a fresh Raspberry Pi OS (see [Setting up the Raspberry Pi](/README.md#setting-up-the-raspberry-pi)) with the same username, and mount your backup drive (or copy the snapshot from the NAS).
1. Reinstall your packages. The list is in the snapshot; review it first, since packages from extra repositories (Docker, Tailscale, etc.) need their repos added before they'll install:
   ```bash
   B=/mnt/sda1/backups/[YOURHOSTNAME]/latest
   less $B/system-state/packages-manual.list
   xargs -a $B/system-state/packages-manual.list sudo apt install -y
   ```
1. Restore data and configs **by folder, not all of `/etc` at once**. Copying the whole of `/etc` over a fresh OS also overwrites files that belong to the new system, and it breaks things if the OS version differs. Instead, copy back the specific config folders for your services (`/etc/pihole`, `/etc/ssh/sshd_config.d`, `/etc/fail2ban/jail.local`, and so on), then your home folder, crontabs and scripts:
   ```bash
   sudo rsync -a $B/home/ /home/pi/
   sudo rsync -a $B/local-sbin/ /usr/local/sbin/
   sudo rsync -a $B/crontabs/ /var/spool/cron/crontabs/
   ```
1. Restore the SSH host keys (`$B/etc/ssh/ssh_host_*`) if you want your computers to recognize the Pi without a "host key changed" warning.
1. For Docker, stop Docker, restore the volumes, then start it and recreate your containers from their compose files (listed in `system-state/docker-containers.list`):
   ```bash
   sudo systemctl stop docker
   sudo rsync -a $B/docker-volumes/ /var/lib/docker/volumes/
   sudo systemctl start docker
   ```
1. Reboot, and compare `systemctl list-unit-files --state=enabled` against `system-state/services-enabled.list` for anything you missed.

Doing this once on a spare SD card — while the original keeps running — is the only way to know your backup is actually complete. It's also the safest way to upgrade to a new Raspberry Pi OS release (see [Upgrading the OS](/README.md#upgrading-to-a-new-os-release)).

## Sources

- https://download.samba.org/pub/rsync/rsync.1
- http://www.mikerubel.org/computers/rsync_snapshots/
- https://linux.die.net/man/1/mountpoint
