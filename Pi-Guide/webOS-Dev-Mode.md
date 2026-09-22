# webOS Dev Mode Renewal

Keep an LG webOS TV's Developer Mode from expiring, so sideloaded apps (like the [Homebrew Channel](https://www.webosbrew.org/) or ad-free YouTube clients) keep working. LG's Developer Mode session lasts at most 1000 hours (about 41 days); when it runs out, the TV turns Developer Mode off and removes your sideloaded apps. Normally you'd have to remember to open the Developer Mode app and press "Extend" every month. This does it daily, automatically.

**The Pi never talks to the TV.** Renewing is a single web request to LG's servers using the TV's session token. The TV can be on a different network, in a different city, or switched off — the Pi only needs internet access. You only need to be on the TV's network once, to read the token.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Get the Session Token](#get-the-session-token)
- [Installation](#installation)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- An LG webOS TV with the **Developer Mode** app installed and Developer Mode turned on (sign in with a free LG Developer account at https://webostv.developer.lge.com).
- A computer on the same network as the TV, with an SSH client, to read the token once. It doesn't have to be the Pi.
- Recommended: [Job Monitoring](/Pi-Guide/Job-Monitoring.md). If this silently stops working, you'll only find out when Developer Mode expires — and fixing that means being in front of the TV.

## Get the Session Token

The token is a file on the TV. You read it over SSH using the key the Developer Mode app provides.

1. In the Developer Mode app on the TV, note the TV's IP address and the **passcode** shown on screen.
1. Get the TV's SSH key onto your computer. Either:
   - If you've already connected the TV with **webOS Dev Manager** (the desktop app), it saved the key in your `~/.ssh` folder under a random hex name like `949d3be52e` — *not* named `webos_...`. Look for the newest file there without a `.pub` extension.
   - Or download it directly: turn on **Key Server** in the Developer Mode app (it turns itself back off, so check it's on each time), then:
     ```bash
     curl -f -o ~/.ssh/webos_rsa http://[TVIPADDRESS]:9991/webos_rsa
     ```
1. Fix the key's permissions — these tools save it readable by everyone, and `ssh` refuses to use a key like that:
   ```bash
   chmod 600 ~/.ssh/[KEYFILE]
   ```
1. Read the token:
   ```bash
   ssh prisoner@[TVIPADDRESS] -p 9922 -i ~/.ssh/[KEYFILE] \
       -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa \
       "cat /var/luna/preferences/devmode_enabled"
   ```
   - The two `ssh-rsa` options are required: the TV's SSH server uses an old key type that modern SSH turns off by default.
   - When asked for the key's passphrase, enter the **passcode from the Developer Mode app**. If you just press Enter, SSH quietly gives up on the key and reports `Permission denied` — which looks like a wrong or outdated key, but isn't.

   It prints a long string of letters and numbers — that's your token. Treat it like a password.

## Installation

1. Save the token on the Pi, readable only by root:
   ```bash
   sudo nano /etc/webos-devmode.token
   ```
   Paste the token in, save, then:
   ```bash
   sudo chmod 600 /etc/webos-devmode.token
   ```
1. Create the script:
   ```bash
   sudo nano /usr/local/sbin/webos-devmode-renew.sh
   ```
   ```bash
   #!/bin/bash
   #
   # Keep an LG webOS TV's developer-mode session from expiring.
   #
   # The renewal is a plain HTTPS GET to LG's servers, keyed on the TV's session
   # token. The TV itself is NEVER contacted -- it does not matter what network or
   # city the TV is on, whether it is powered on, or whether it is on Tailscale.
   # This box only needs outbound internet.
   #
   # The token is read off the TV once and saved to $TOKEN_FILE -- see the
   # guide. It stays valid as long as the session never lapses; if dev mode
   # expires and is re-enabled, LG issues a new token and it must be re-read.
   #
   # Usage:
   #   webos-devmode-renew.sh           renew, then report remaining time
   #   webos-devmode-renew.sh --check   report remaining time only, do not renew
   #
   # Exit codes: 0 ok (or not yet configured), 1 renewal/check failed.

   set -uo pipefail

   TOKEN_FILE=/etc/webos-devmode.token
   API=https://developer.lge.com/secure
   CONF=/etc/job-monitoring.conf   # optional, see Job Monitoring guide

   # Sessions cap at 1000 h. Warn if a renewal leaves us under this many hours,
   # which means the reset is being accepted but not actually extending.
   WARN_BELOW_HOURS=240

   log() { printf '%s %s\n' "$(date '+%Y-%m-%d %H:%M:%S')" "$*"; }

   # First and last lines of every run. The login status message judges only the
   # log's last line:
   # "exit=N" is a finished run, anything else means the run was killed partway.
   # SIGKILL/power loss skip the trap by nature; TERM/HUP/INT must be trapped
   # explicitly or bash reports the killed run as exit=0.
   log "run: start${1:+ $1}"
   trap 'log "run: exit=$?"' EXIT
   trap 'exit 129' HUP; trap 'exit 130' INT; trap 'exit 143' TERM

   # healthchecks.io dead-man's switch. No URL configured => silently disabled.
   # Alerts both on failure and on the job silently ceasing to run at all.
   [[ -r $CONF ]] && . "$CONF"
   hc() {
       local url="${HC_WEBOS_URL:-}"
       [[ -z $url ]] && return 0
       curl -fsS -m 10 --retry 3 -o /dev/null "${url}${1:-}" 2>/dev/null || true
   }
   die() { hc /fail; exit 1; }

   # Pull one string field out of LG's flat JSON reply.
   field() { sed -n "s/.*\"$2\":\"\([^\"]*\)\".*/\1/p" <<<"$1"; }

   api() {
       curl --silent --show-error --max-time 30 --retry 3 --retry-delay 10 \
            "$API/$1.dev?sessionToken=$TOKEN"
   }

   # Report the session state, setting REMAINING_HOURS. Returns 1 if LG rejected
   # the token.
   REMAINING_HOURS=
   report() {
       local what=$1 body result msg
       REMAINING_HOURS=
       body=$(api CheckDevModeSession) || { log "$what: could not reach LG"; return 1; }
       result=$(field "$body" result)
       msg=$(field "$body" errorMsg)
       if [[ $result != success ]]; then
           log "$what: LG rejected the token -- $(field "$body" errorCode) $msg"
           return 1
       fi
       log "$what: ${msg} remaining (h:mm:ss)"
       REMAINING_HOURS=${msg%%:*}
   }

   if [[ ! -s $TOKEN_FILE ]]; then
       log "not configured: $TOKEN_FILE is missing or empty, nothing to renew"
       exit 0
   fi

   TOKEN=$(grep -v '^[[:space:]]*#' "$TOKEN_FILE" | tr -d '[:space:]')
   if [[ -z $TOKEN || $TOKEN == PUT_YOUR_SESSION_TOKEN_HERE ]]; then
       log "not configured: no session token in $TOKEN_FILE, nothing to renew"
       exit 0
   fi

   if [[ ${1:-} == --check ]]; then
       report "session" || exit 1
       exit 0
   fi

   hc /start
   report "before" || die

   body=$(api ResetDevModeSession) || { log "renew: could not reach LG"; die; }
   if [[ $(field "$body" result) != success ]]; then
       log "renew: FAILED -- $(field "$body" errorCode) $(field "$body" errorMsg)"
       die
   fi
   log "renew: accepted"

   report "after" || die

   # Guard the comparison: bash arithmetic coerces a non-numeric string to 0, so an
   # unexpected reply from LG would otherwise look like "0h left" and warn falsely.
   if [[ $REMAINING_HOURS =~ ^[0-9]+$ ]]; then
       if [[ $REMAINING_HOURS -lt $WARN_BELOW_HOURS ]]; then
           log "WARNING: only ${REMAINING_HOURS}h left after a successful renewal -- the reset is"
           log "WARNING: being accepted but not extending; check the session on the TV"
           die
       fi
   elif [[ -n $REMAINING_HOURS ]]; then
       log "WARNING: could not parse remaining hours from '${REMAINING_HOURS}'"
       die
   fi

   hc
   exit 0
   ```
   ```bash
   sudo chmod 700 /usr/local/sbin/webos-devmode-renew.sh
   ```
1. Schedule it daily in root's crontab:
   ```bash
   sudo crontab -e
   ```
   ```
   30 3 * * * /usr/local/sbin/webos-devmode-renew.sh >> /var/log/webos-devmode-renew.log 2>&1
   ```
1. Set up log rotation, and add a healthchecks.io check with `HC_WEBOS_URL` in `/etc/job-monitoring.conf`, as described in [Job Monitoring](/Pi-Guide/Job-Monitoring.md).

The script fails (and alerts, if you set up monitoring) when LG rejects the token, when LG can't be reached, or when LG accepts a renewal but the time left is still under 240 hours — which means renewals are being "accepted" without actually extending anything.

## Testing

Check how much time is left, without renewing:

```bash
sudo /usr/local/sbin/webos-devmode-renew.sh --check
```

You should see something like `session: 997:29:10 remaining (h:mm:ss)`. Then run a real renewal and check the time went back up to about 1000 hours:

```bash
sudo /usr/local/sbin/webos-devmode-renew.sh
```

The Developer Mode app on the TV shows the same remaining time.

## Troubleshooting

- **"LG rejected the token":** the session expired at some point, or Developer Mode was turned off and on again. Either way LG issued a **new** token — repeat [Get the Session Token](#get-the-session-token) from the TV's network and replace the contents of `/etc/webos-devmode.token`.
- **"could not reach LG":** the Pi has no internet access, or LG's site is down. One failed day is harmless — there are weeks of margin — but look into it if it keeps happening.
- **`Permission denied (publickey)` when reading the token:** see the passphrase note above first. If the key really is out of date, download a fresh one from the Key Server.
- **SSH says `no matching host key type found`:** you left out the `-o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa` options.

If you back up `/etc` (e.g. with [Snapshot Backups](/Pi-Guide/Snapshot-Backups.md)), the token goes into your backup too — which is what you want for a restore, but keep it in mind for who can read the backup.

## Sources

- https://webostv.developer.lge.com/develop/getting-started/developer-mode-app
- https://www.webosbrew.org/
- https://github.com/webosbrew/dev-manager-desktop
