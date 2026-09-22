# Immich Backup (BorgBackup)

Encrypted, deduplicated, versioned backups of your [immich](/Pi-Guide/immich.md) photos and database, sent to a NAS (or any other drive) every night.

An Immich backup needs two things: the **photo library** and the **database**, which holds your albums, faces, people, and which photo is which. The library alone doesn't restore as a working Immich, and copying the database's files while Postgres is running produces a corrupt copy. So this guide dumps the database to a file first, then backs up the library and the dump together with [BorgBackup](https://www.borgbackup.org/).

Why borg rather than plain rsync:

- **Versioned.** Keeps daily, weekly and monthly archives, so a photo deleted by mistake, or a database broken by a bad update, can still be recovered weeks later.
- **Deduplicated.** Each archive only stores what changed. A nightly backup of a 500 GB library that gained 20 new photos adds about 20 photos' worth of data.
- **Encrypted.** The NAS only ever sees encrypted chunks. Someone with access to the NAS can't browse your photos without the passphrase, which stays on the Pi.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Create the Repository](#create-the-repository)
- [The Backup Script](#the-backup-script)
- [Schedule It](#schedule-it)
- [Testing](#testing)
- [Restoring](#restoring)
- [Sources](#sources)

## Prerequisites

- [immich](/Pi-Guide/immich.md), installed with Docker Compose. This guide assumes it's in `/mnt/sda1/immich-app` with the default container names and `DB_USERNAME=postgres`.
- Somewhere to back up to, mounted on the Pi. This guide uses a NAS folder mounted at `/mnt/nas` over NFS. A second USB drive works the same way. **Not** the same drive Immich lives on, or one drive failure takes both.
- Recommended: [Job Monitoring](/Pi-Guide/Job-Monitoring.md), so you hear about a failed backup when it happens, not when you need the backup.

## Installation

1. Install borg:
   ```bash
   sudo apt install borgbackup
   borg --version
   ```
   This guide needs borg 1.2 or newer, for `borg compact`. Raspberry Pi OS Bookworm ships 1.2.
1. If you're backing up to a NAS over NFS, mount it on boot. Open `/etc/fstab`:
   ```bash
   sudo nano /etc/fstab
   ```
   and add a line like this, with your NAS's address and export path:
   ```
   192.168.50.20:/volume1/backups /mnt/nas nfs rw,_netdev,vers=4,hard,noatime,x-systemd.automount,nofail 0 0
   ```
   - `x-systemd.automount` mounts the share the first time something uses it, rather than at boot, and `nofail` means a NAS that's switched off can't stop the Pi from booting.
   - Then `sudo findmnt --verify`, `sudo systemctl daemon-reload`, `sudo mkdir -p /mnt/nas`, and `ls /mnt/nas` to trigger the mount.

## Create the Repository

A borg *repository* is the folder on the NAS that holds all your archives.

1. Create the passphrase. It's stored in a root-only file so the nightly backup can run unattended:
   ```bash
   sudo sh -c 'umask 077; head -c 24 /dev/urandom | base64 > /root/.borg-passphrase'
   sudo cat /root/.borg-passphrase
   ```
   <ins>IMPORTANT:</ins> Copy the passphrase into your password manager **now**. Without it the backup can't be decrypted. That's no problem while the Pi is alive, but if the Pi's SD card dies, the file dies with it and your backup is unreadable. The passphrase is deliberately *not* stored on the NAS, so that access to the NAS alone isn't enough to read your photos.
1. Create the repository:
   ```bash
   sudo mkdir -p /mnt/nas/backup_immich
   sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg init --encryption=repokey /mnt/nas/backup_immich/immich-borg
   ```
   `repokey` stores the encryption key inside the repository, protected by your passphrase.
1. Make it readable by root only:
   ```bash
   sudo chmod 700 /mnt/nas/backup_immich/immich-borg
   ```
   If this fails with "Operation not permitted", your NAS maps root to an unprivileged user ("root squash"). Set permissions on the NAS side instead.

**Already have a repository with no passphrase** (created with `--encryption=repokey` and an empty passphrase)? You don't need a new repository or a full re-upload. This re-wraps the existing key in seconds:

```bash
sudo borg key change-passphrase /mnt/nas/backup_immich/immich-borg
```

Enter an empty current passphrase, then the new one (the contents of `/root/.borg-passphrase`).

## The Backup Script

1. Create the script:
   ```bash
   sudo nano /usr/local/sbin/immich-borg-backup.sh
   ```
1. Paste the following in, and check the paths at the top:
   ```bash
   #!/bin/bash
   # Nightly Immich backup: dump the database, then borg the library + dump.
   set -uo pipefail

   IMMICH_DIR="/mnt/sda1/immich-app"
   LIBRARY="$IMMICH_DIR/library"            # UPLOAD_LOCATION from Immich's .env
   DUMP_DIR="$IMMICH_DIR/db-dump"           # outside the library, see notes
   REPO="/mnt/nas/backup_immich/immich-borg"
   DB_USER="postgres"                       # DB_USERNAME from Immich's .env
   CONF="/etc/job-monitoring.conf"          # optional, see Job Monitoring guide

   export BORG_PASSCOMMAND="cat /root/.borg-passphrase"
   FAILED=0

   log() { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }

   log "start"
   trap 'log "exit=$?"' EXIT
   trap 'exit 129' HUP; trap 'exit 130' INT; trap 'exit 143' TERM

   [[ -r $CONF ]] && . "$CONF"
   hc() {
       local url="${HC_IMMICH_BORG_URL:-}"
       [[ -z $url ]] && return 0
       curl -fsS -m 10 --retry 3 -o /dev/null "${url}${1:-}" 2>/dev/null || true
   }

   hc /start

   # 1. Dump the database. No -t on docker exec: a TTY would mix error
   #    messages into the dump and turn every line ending into CRLF.
   mkdir -p "$DUMP_DIR"
   if docker exec immich_postgres pg_dumpall --clean --if-exists --username="$DB_USER" \
           > "$DUMP_DIR/immich_db_dump.sql.tmp" \
       && mv "$DUMP_DIR/immich_db_dump.sql.tmp" "$DUMP_DIR/immich_db_dump.sql"; then
       log "database dump OK ($(du -h "$DUMP_DIR/immich_db_dump.sql" | cut -f1))"
   else
       log "FAILED: database dump"
       rm -f "$DUMP_DIR/immich_db_dump.sql.tmp"
       FAILED=1
   fi

   # 2. Back up library + dump. thumbs/ and encoded-video/ are regenerated by
   #    Immich; backups/ holds Immich's own automatic DB dumps, which duplicate ours.
   if borg create --stats --compression lz4 "$REPO::{now}" "$LIBRARY" "$DUMP_DIR" \
           --exclude "$LIBRARY/thumbs/" \
           --exclude "$LIBRARY/encoded-video/" \
           --exclude "$LIBRARY/backups/"; then
       log "borg create OK"
   else
       log "FAILED: borg create"
       FAILED=1
   fi

   # 3. Retention: 7 daily, 4 weekly, 3 monthly. compact actually frees the space.
   if borg prune --keep-daily=7 --keep-weekly=4 --keep-monthly=3 "$REPO" \
       && borg compact "$REPO"; then
       log "prune + compact OK"
   else
       log "FAILED: prune/compact"
       FAILED=1
   fi

   if (( FAILED == 0 )); then hc; else hc /fail; fi
   exit "$FAILED"
   ```
1. Make it root-only:
   ```bash
   sudo chmod 700 /usr/local/sbin/immich-borg-backup.sh
   ```

Notes on the script:

- **The dump lives outside the library folder** (`db-dump/`, next to `library/`), so Immich's folders only ever hold what Immich itself put there, and the dump is easy to find at restore time.
- **The dump is written to a `.tmp` file first**, then renamed. If a dump fails halfway, the previous good dump stays in place.
- **Each step keeps going if an earlier one fails**, so a broken database dump still gets you a backup of the photos. But the `FAILED` flag makes the script **exit non-zero** at the end, which sends the `/fail` ping. Without the flag, the failures would only be written to the log and healthchecks.io would get a success ping. See [Job Monitoring](/Pi-Guide/Job-Monitoring.md#monitoring-a-script).
- **Borg won't write to the SD card if the NAS isn't mounted.** `borg create` needs an existing repository, so if the share isn't there it fails with "Repository does not exist" rather than quietly filling the SD card. It's reported as a failure like any other.
- **Don't enable borg's `append_only` mode on this repository.** It sounds like good ransomware protection, but in append-only mode `prune` and `compact` are only recorded, not carried out, so the repository grows forever until you clean it up by hand. Protecting against a compromised Pi needs a different design, like a NAS-side snapshot of the backup folder.

## Schedule It

Run it nightly from **root's** crontab:

```bash
sudo crontab -e
```

```
0 3 * * * /usr/local/sbin/immich-borg-backup.sh >> /var/log/immich-borg-backup.log 2>&1
```

Then add log rotation and a healthchecks.io check (`HC_IMMICH_BORG_URL` in `/etc/job-monitoring.conf`) as described in [Job Monitoring](/Pi-Guide/Job-Monitoring.md). Give the check a generous grace time: the first run uploads the entire library and can take many hours.

Borg commands need the passphrase. When you run one by hand, pass it the same way the script does, or borg will prompt for it and look like it's hanging:

```bash
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list /mnt/nas/backup_immich/immich-borg
```

## Testing

1. Run it once by hand. The first run copies everything, so start it somewhere that won't be interrupted, like `tmux`:
   ```bash
   sudo /usr/local/sbin/immich-borg-backup.sh 2>&1 | sudo tee -a /var/log/immich-borg-backup.log
   ```
   The log should end with `exit=0`.
1. Check that the archive has real content. After the first run, later backups often **finish in seconds**. That's normal: borg skips files it has already stored. Judge a run by its numbers, not by how long it took:
   ```bash
   R=/mnt/nas/backup_immich/immich-borg
   sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg info --last 1 $R
   ```
   "Number of files" should be about your photo and video count. "Deduplicated size" is what this run actually added.
1. Check that the database dump and the excludes came out right:
   ```bash
   sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list --short $R::$(sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg list --last 1 --short $R) | grep -E 'immich_db_dump|/thumbs/|/encoded-video/' | head
   ```
   You should see `immich_db_dump.sql`, and `thumbs`/`encoded-video` only as bare folder names with nothing inside them.
1. Confirm the passphrase is really required. This should fail with a passphrase error:
   ```bash
   sudo BORG_PASSPHRASE="" borg list $R
   ```
1. Every few months, check the repository for damage. `--verify-data` reads back and decrypts every chunk, so it takes as long as a full restore:
   ```bash
   sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg check --verify-data $R
   ```

## Restoring

### A few photos, or browsing old archives

Mount the repository as a normal folder, copy out what you need, and unmount:

```bash
R=/mnt/nas/backup_immich/immich-borg
sudo mkdir -p /mnt/borg
sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg mount $R /mnt/borg
sudo ls /mnt/borg                       # one folder per archive, named by date
# copy what you need out of /mnt/borg/<archive>/mnt/sda1/immich-app/library/...
sudo borg umount /mnt/borg
```

Photos copied back this way are just files. Re-upload them through the Immich app or web UI so Immich knows about them.

### The whole Immich instance

This replaces Immich's database with the backup. Anything added to Immich since that backup is lost, so be sure it's what you want. It's also the procedure for moving to a new Pi. There, install Docker and borg first, restore `/root/.borg-passphrase` from your password manager, and mount the NAS.

1. Stop Immich and move the old database folder aside, rather than deleting it, until the restore has worked:
   ```bash
   cd /mnt/sda1/immich-app
   docker compose down
   sudo mv postgres postgres.old          # DB_DATA_LOCATION from .env
   ```
1. Restore the library and the dump from the archive you want. borg extracts into the current directory, using the full original paths:
   ```bash
   R=/mnt/nas/backup_immich/immich-borg
   cd /
   sudo BORG_PASSCOMMAND="cat /root/.borg-passphrase" borg extract $R::[ARCHIVENAME] mnt/sda1/immich-app/library mnt/sda1/immich-app/db-dump
   ```
1. **Recreate the marker files for the folders that weren't backed up.** At startup, Immich checks for a hidden `.immich` file in each of its folders and refuses to start if one it previously wrote is missing. `thumbs/`, `encoded-video/` and `backups/` were excluded, so their markers weren't backed up either:
   ```bash
   cd /mnt/sda1/immich-app/library
   for d in thumbs encoded-video backups; do sudo mkdir -p $d && sudo touch $d/.immich; done
   ```
1. Start only the database, and load the dump into it. The `sed` fixes a `search_path` setting in `pg_dumpall` output that Immich's database needs, as in Immich's own restore instructions:
   ```bash
   cd /mnt/sda1/immich-app
   docker compose pull && docker compose create
   docker start immich_postgres && sleep 10
   sed "s/SELECT pg_catalog.set_config('search_path', '', false);/SELECT pg_catalog.set_config('search_path', 'public, pg_catalog', true);/g" \
       db-dump/immich_db_dump.sql | docker exec -i immich_postgres psql --dbname=postgres --username=postgres
   ```
1. Start everything:
   ```bash
   docker compose up -d
   ```
1. Log in, check that your photos, albums and people are there, then go to **Administration → Jobs** and run **Generate Thumbnails** and **Transcode Videos** (both "Missing") to rebuild what wasn't backed up.
1. Once everything checks out, delete the old database folder: `sudo rm -rf /mnt/sda1/immich-app/postgres.old`.

Doing a full restore once, on a spare Pi or SD card while the real one keeps running, is the only way to know the backup actually works.

## Sources

- https://borgbackup.readthedocs.io/en/stable/quickstart.html
- https://borgbackup.readthedocs.io/en/stable/usage/key.html
- https://borgbackup.readthedocs.io/en/stable/usage/notes.html#append-only-mode-forbid-compaction
- https://docs.immich.app/administration/backup-and-restore
- https://docs.immich.app/administration/system-integrity
