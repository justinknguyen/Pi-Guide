# Job Monitoring

Get alerted when a scheduled job (a backup, a sync, a renewal script) fails — **or silently stops running**. Cron doesn't tell you anything: on a Pi there's usually no email set up, so a job that breaks just writes errors to a log nobody reads, and you find out months later when you need the backup.

This guide uses [healthchecks.io](https://healthchecks.io) (free for up to 20 jobs), a "dead man's switch": your job pings a URL each time it runs, and healthchecks.io alerts you when the pings **stop arriving**. That catches the failures a script can't report itself — cron not running, the script deleted, the Pi offline or out of power.

It also adds a status message shown every time you SSH in, covering the same jobs plus any pending reboot.

## Table of Contents

- [Create the Checks](#create-the-checks)
- [Store the Ping URLs](#store-the-ping-urls)
- [Monitoring a Script](#monitoring-a-script)
- [Monitoring a One-Line Cron Job](#monitoring-a-one-line-cron-job)
- [Keep the Logs from Growing](#keep-the-logs-from-growing)
- [Login Status Message](#login-status-message)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Create the Checks

1. Sign up at https://healthchecks.io and create a project.
1. Click "Add Check" for each job, then "Edit" its schedule:
   - Choose **Cron** (not "Simple"), paste in the same schedule as your crontab line (e.g. `30 4 * * *`), and set the **time zone** to the Pi's (`timedatectl | grep zone`). A Cron check also flags a job that runs at the *wrong time*, not just one that stops entirely.
   - Set the **grace time** to a bit longer than the job normally takes (e.g. 1 hour for a backup).
1. Under "Integrations", set up how you want to be alerted (email is on by default; phone apps, Discord, Telegram, ntfy and others are available).
1. Copy each check's ping URL (`https://hc-ping.com/...`).

## Store the Ping URLs

Treat ping URLs like passwords — anyone who has one can fake your job's pings. Keep them in one root-only file rather than inside your scripts or crontab:

```bash
sudo nano /etc/job-monitoring.conf
```

```bash
# healthchecks.io ping URLs. Leave a value empty to turn off pinging for that job.
HC_BACKUP_URL=https://hc-ping.com/your-uuid-here
HC_MYJOB_URL=https://hc-ping.com/another-uuid-here
```

```bash
sudo chmod 600 /etc/job-monitoring.conf
```

## Monitoring a Script

Use this template for any job you write as a script. Put your job's commands in the marked section:

```bash
sudo nano /usr/local/sbin/myjob.sh
```

```bash
#!/bin/bash
# Template for a monitored cron job.
set -uo pipefail

CONF=/etc/job-monitoring.conf
URL_VAR=HC_MYJOB_URL        # which line of $CONF holds this job's ping URL

log() { printf '[%s] %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }

# First and last lines of every run. The login message judges each job by
# the LAST line of its log: "exit=N" means it finished; anything else means
# it was killed partway. Bash doesn't run the EXIT trap with the right code
# on TERM/HUP/INT unless they're trapped too.
log "start"
trap 'log "exit=$?"' EXIT
trap 'exit 129' HUP; trap 'exit 130' INT; trap 'exit 143' TERM

# healthchecks.io ping. An empty or missing URL just disables pinging.
# ${1:-} (not $1) because the success ping has no argument and set -u
# would otherwise abort the script at the very end. "|| true" so a
# healthchecks.io outage can never fail the job itself.
[[ -r $CONF ]] && . "$CONF"
hc() {
    local url="${!URL_VAR:-}"
    [[ -z $url ]] && return 0
    curl -fsS -m 10 --retry 3 -o /dev/null "${url}${1:-}" 2>/dev/null || true
}

hc /start

# ---- your job goes here ---------------------------------------------------
if ! rsync -a /home/pi/documents/ /mnt/sda1/documents/; then
    log "FAILED: rsync of documents"
    hc /fail
    exit 1
fi
# ---------------------------------------------------------------------------

log "completed OK"
hc
```

```bash
sudo chmod 700 /usr/local/sbin/myjob.sh
```

What it does:

- Pings `/start` when it begins, the plain URL on success, and `/fail` on failure. The `/start` ping also lets healthchecks.io spot a run that started but never finished, and shows how long each run took.
- Writes a `start` line first and an `exit=N` line last, even if the script is stopped partway with `kill` or `Ctrl+C`. The [login status message](#login-status-message) relies on this.
- If the job fails, `exit 1` makes that visible to anything else that checks the result.

Two details in `hc()` look fussy but are needed:

- `${1:-}` instead of `$1`: the success ping is called with no argument, and with `set -u`, plain `$1` would crash the script on the very last line — after the job itself succeeded, so healthchecks.io would alert you about a job that worked.
- `|| true`: if healthchecks.io itself is down or the Pi's internet is out, the job still runs and still succeeds.

If you adapt this for a script that uses `set -e`, don't shorten `hc()` to a one-liner like `[[ -n $url ]] && curl ... "$url$1"`. When the URL is empty that line returns 1, and when healthchecks.io is unreachable curl returns 7. Under `set -e`, either one stops the script at `hc /start`, **before the job runs**, and it fails silently because monitoring is the thing that broke. Keep the explicit `return 0` and `|| true`.

A script with several steps (dump a database, then back it up, then prune) shouldn't stop at the first failure, but it still has to *end* with a non-zero exit. Otherwise the failure is only in the log and healthchecks.io gets a success ping. Track it with a flag:

```bash
FAILED=0
step_one || { log "FAILED: step one"; FAILED=1; }
step_two || { log "FAILED: step two"; FAILED=1; }
exit "$FAILED"
```

Schedule it with `sudo crontab -e` (root's crontab, since the config file is root-only), sending all output to a log:

```
30 4 * * * /usr/local/sbin/myjob.sh >> /var/log/myjob.log 2>&1
```

## Monitoring a One-Line Cron Job

For a job that's a single command in your crontab (like the one in [Rclone](/Pi-Guide/Rclone.md)), you can skip the script and ping directly from the cron line:

```
0 0 * * * rclone copy [FOLDERDIRECTORY] "gdrive:backups" --log-file /home/pi/rclone.log && curl -fsS -m 10 --retry 3 -o /dev/null https://hc-ping.com/your-uuid-here || curl -fsS -m 10 --retry 3 -o /dev/null https://hc-ping.com/your-uuid-here/fail
```

This sends the success ping if the command worked and `/fail` if it didn't. It doesn't send `/start`, and the ping URL sits in your crontab instead of the root-only file, but for a personal cron job that's usually fine.

## Keep the Logs from Growing

Logs appended to by cron grow forever unless something rotates them. Create a rotation rule:

```bash
sudo nano /etc/logrotate.d/myjob
```

```
/var/log/myjob.log {
    monthly
    rotate 6
    compress
    delaycompress
    missingok
    notifempty
}
```

This keeps 6 months of history. Check it with `sudo logrotate -d /etc/logrotate.d/myjob`.

## Login Status Message

healthchecks.io tells you when something breaks. This shows the current state every time you SSH in, plus two things that are otherwise invisible: a reboot waiting to apply updates (see [Unattended-Upgrades](/Pi-Guide/Unattended-Upgrades.md#pending-reboots)), and staged bootloader firmware.

1. Create the script. Scripts in `/etc/update-motd.d/` run at each SSH login:
   ```bash
   sudo nano /etc/update-motd.d/95-job-status
   ```
1. Paste the following in, and edit the `JOBS` list to match your own jobs — one line each, with its log file, script name, and how many days without a run before it should warn you:
   ```bash
   #!/bin/bash
   # Login status: pending reboots and the last result of each monitored job.
   # Reads local files only, so it adds no noticeable delay to logging in.

   # <name>|<log file>|<script name>|<days before a success counts as stale>
   JOBS=(
       "backup|/var/log/backup.log|backup.sh|2"
   )

   RED=$'\033[31m'; YEL=$'\033[33m'; GRN=$'\033[32m'; OFF=$'\033[0m'

   [[ -f /var/run/reboot-required ]] &&
       echo "  ${RED}REBOOT REQUIRED${OFF} for: $(tr '\n' ' ' < /var/run/reboot-required.pkgs 2>/dev/null)"
   [[ -f /boot/firmware/pieeprom.upd ]] &&
       echo "  ${RED}REBOOT REQUIRED${OFF} to flash staged bootloader firmware"

   for job in "${JOBS[@]}"; do
       IFS='|' read -r name log script maxage <<< "$job"
       if [[ ! -r $log ]]; then
           echo "  $name: ${YEL}no log yet${OFF}"; continue
       fi
       age=$(( ($(date +%s) - $(stat -c %Y "$log")) / 86400 ))
       # Judge ONLY the last line. Don't search the log for a success message:
       # a killed run leaves the previous run's success line just above it.
       last=$(grep -v '^[[:space:]]*$' "$log" | tail -1)
       if [[ $last =~ exit=([0-9]+)$ ]]; then
           code=${BASH_REMATCH[1]}
           if [[ $code != 0 ]]; then
               echo "  $name: ${RED}FAILED${OFF} (exit $code, ${age}d ago) - see $log"
           elif (( age > maxage )); then
               echo "  $name: ${YEL}last ran ${age}d ago${OFF} - is cron running?"
           else
               echo "  $name: ${GRN}ok${OFF} (${age}d ago)"
           fi
       elif pgrep -f "/$script" >/dev/null; then
           echo "  $name: ${YEL}running now${OFF}"
       else
           echo "  $name: ${RED}INTERRUPTED${OFF} - last run never finished (${age}d ago)"
       fi
   done
   echo
   ```
1. Make it executable, and try it:
   ```bash
   sudo chmod +x /etc/update-motd.d/95-job-status
   sudo /etc/update-motd.d/95-job-status
   ```

Each job is judged by the **last line of its log only**, which is why the template above always ends with `exit=N`:

| Last line | Shown as |
| --- | --- |
| `exit=0`, recent | ok |
| `exit=0`, older than the stale limit | "last ran Nd ago" — cron may have stopped |
| `exit=` anything else | FAILED |
| anything else, script still running | running now |
| anything else, script not running | INTERRUPTED — killed partway (power loss, out of memory, `kill -9`) |

Don't be tempted to simplify this to "search the log for a success message." Logs are appended, so when a run is killed partway, the *previous* run's success message is sitting just above it — and the check reports a dead job as ok.

Keep this script to local file reads (no network calls, no `apt` or `docker` commands). It runs on every login, so anything slow here delays every SSH session.

## Testing

1. Run the job by hand and check that the log ends in `exit=0` and the check turns green on healthchecks.io:
   ```bash
   sudo /usr/local/sbin/myjob.sh >> /var/log/myjob.log 2>&1
   tail -3 /var/log/myjob.log
   ```
1. Confirm alerts actually reach you — this is the step people skip, and it's the one that matters. Send a failure ping by hand:
   ```bash
   sudo bash -c '. /etc/job-monitoring.conf && curl -fsS "$HC_MYJOB_URL/fail"'
   ```
   You should get an alert within a minute or two. Run the job again afterwards to turn the check green.
1. Log out and back in to see the status message.

## Troubleshooting

**Every run shows as "late" or down on healthchecks.io, but the logs say the job succeeded.** Check each check's schedule time zone. A Cron check left at the default `UTC` expects pings hours away from when your Pi, on local time, actually sends them, so every run falls outside the grace time. Set it to the Pi's zone (`timedatectl | grep zone`). Nothing on the Pi needs to change.

**The check never goes green, and the job's log has no errors.** The ping URL is probably wrong. A typo'd or empty URL is silently ignored on purpose, so monitoring can't break the job. Send a ping by hand to test it. This should print `OK`:
```bash
sudo bash -c '. /etc/job-monitoring.conf && curl -fsS "$HC_MYJOB_URL"'
```

**The login message says "INTERRUPTED" for a job you just switched to this template.** Its log was written by the old script, so it has no `exit=` line yet, and that looks the same as a killed run. It clears after the first complete run. Run the job once by hand if you don't want to wait.

## Sources

- https://healthchecks.io/docs/
- https://healthchecks.io/docs/bash/
- https://healthchecks.io/docs/configuring_checks/
- https://manpages.debian.org/bookworm/libpam-modules/pam_motd.8.en.html
