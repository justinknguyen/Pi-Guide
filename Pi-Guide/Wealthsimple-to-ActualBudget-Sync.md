# Wealthsimple to Actual Budget Sync

Automate syncing Wealthsimple transactions into ActualBudget using a Python script on your Raspberry Pi, via Wealthsimple's GraphQL API.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Upgrading ws-api](#upgrading-ws-api)
- [Configuration](#configuration)
- [Testing](#testing)
- [Automation](#automation)
- [Troubleshooting](#troubleshooting)
- [Python Script](#python-script)
- [Sources](#sources)

## Prerequisites

- [Actual Budget](/Pi-Guide/Actual-Budget.md) — a running Actual server to import into
- [Docker](/Pi-Guide/Docker.md) (if running ActualBudget in a container)

## Installation

1. Update system packages:
   ```bash
   sudo apt update && sudo apt upgrade
   ```
1. Install Python and pip:
   ```bash
   sudo apt install python3 python3-venv python3-pip -y
   ```
1. Create and activate a virtual environment:
   ```bash
   python3 -m venv ~/actual_env
   source ~/actual_env/bin/activate
   ```
1. Install required dependencies:
   ```bash
    pip install "ws-api>=0.38.1" actualpy keyring python-dateutil pyotp
   ```
    - `ws-api` ≤0.33.0 has a bug where session refresh silently never fires (see [Troubleshooting](#troubleshooting)) — make sure you're on 0.38.1+.
1. Create the `ws_to_actual.py` script in your home directory (full code in [Python Script](#python-script) below):
   ```bash
   nano ~/ws_to_actual.py
   ```
   - Paste in the script, save with `Ctrl+X` then `Y`, then make it executable:
     ```bash
     chmod +x ~/ws_to_actual.py
     ```

## Upgrading ws-api

To upgrade `ws-api` in the virtual environment used by the sync script:

```bash
source ~/actual_env/bin/activate
python -m pip install --upgrade "ws-api>=0.38.1"
python -m pip show ws-api
```

Confirm that the installed version is `0.38.1` or newer, then test the sync manually:

```bash
python ~/ws_to_actual.py
```

The cron wrapper uses the same virtual environment, so scheduled runs will use the upgraded version automatically.

## Configuration

1. Create a dedicated cron environment file in your home directory, then apply it to the current shell:
   ```bash
   cat > /home/pi/ws_to_actual_env.sh <<'EOF'
   export ACTUAL_BASE_URL='http://localhost:5006'
   export ACTUAL_PASSWORD='your_actual_password'
   export ACTUAL_BUDGET_FILE='Wealthsimple Import'
   export WS_USERNAME='your_ws_email'
   export WS_PASSWORD='your_ws_pass'
   export WS_TOTP_SECRET='your_ws_totp_secret'
   EOF
   chmod 600 /home/pi/ws_to_actual_env.sh
   source /home/pi/ws_to_actual_env.sh
   ```
   - Keep this file the **only** place your credentials live. If you want them loaded automatically for manual test runs, add a line to `~/.bashrc` that *sources* the file, rather than copying the `export` lines into `.bashrc`:
     ```bash
     echo '[ -f ~/ws_to_actual_env.sh ] && . ~/ws_to_actual_env.sh' >> ~/.bashrc
     ```
     `.bashrc` is readable by everyone (mode 644) and gets copied around by backups and dotfile syncs, so passwords pasted straight into it end up exposed in places you didn't intend.
   - Any backup copy you make of a file holding secrets (e.g. `cp ~/.bashrc ~/.bashrc.bak`) needs `chmod 600` too. A plain `cp` creates the new file with default permissions, readable by everyone.
1. Run the script manually the first time:
   ```bash
   source ~/actual_env/bin/activate
   python ~/ws_to_actual.py
   ```
   - You'll be prompted for your Wealthsimple login and OTP (stored securely in your system keyring) if not set in the environment
   - The first run may also ask for your Actual password if not set in the environment
1. (Optional) Limit imported accounts by editing the filter list in your script:
   ```python
   ALLOWED_ACCOUNTS = ["Cash", "Credit card"]
   ```

## Testing

You can test the script manually before automating:

```bash
source ~/actual_env/bin/activate
python ~/ws_to_actual.py
```

The script logs to your terminal. A successful run will look like:

```
INFO Found existing Wealthsimple session in keyring.
INFO Filtered transactions from 273 → 47 (accounts: Cash, Credit card)
INFO Committed 47 transactions to Actual.
INFO Done.
```

## Automation

1. Create a small wrapper script so cron uses a dedicated environment file:
   ```bash
   cat > /home/pi/run_ws_to_actual.sh <<'EOF'
   #!/bin/bash
   set -e

   # Optional healthchecks.io monitoring -- see the Job Monitoring guide.
   MONITORING_CONF="/home/pi/.job-monitoring.conf"
   [ -r "$MONITORING_CONF" ] && source "$MONITORING_CONF"

   # A missing URL or an unreachable healthchecks.io must never fail the job.
   hc_ping() {
       [ -n "${HC_WS_TO_ACTUAL_URL:-}" ] || return 0
       curl -fsS -m 10 --retry 3 -o /dev/null "${HC_WS_TO_ACTUAL_URL}${1:-}" || true
   }

   # Runs on every exit, including a set -e abort: sends the final ping and
   # writes the "exit=N" line the login status message reads.
   on_exit() {
       local rc=$?
       if [ "$rc" -eq 0 ]; then hc_ping; else hc_ping /fail; fi
       echo "[$(date '+%Y-%m-%d %H:%M:%S')] exit=$rc"
   }
   trap on_exit EXIT
   trap 'exit 129' HUP; trap 'exit 130' INT; trap 'exit 143' TERM

   hc_ping /start
   source /home/pi/ws_to_actual_env.sh
   # A normal run takes seconds; the timeout stops a hung Wealthsimple/Actual
   # call from lingering until the next run (exits 124, which pings /fail).
   timeout 10m /home/pi/actual_env/bin/python /home/pi/ws_to_actual.py
   EOF
   chmod +x /home/pi/run_ws_to_actual.sh
   ```
   - This avoids cron failing when `~/.bashrc` is not sourced or contains interactive-only checks. Calling the venv's `python` directly is enough; the venv doesn't need to be activated.
   - The monitoring part is optional and does nothing until you create the config file. To turn it on, create a check as described in [Job Monitoring](/Pi-Guide/Job-Monitoring.md#create-the-checks) (Cron schedule `0 */12 * * *`), then save its ping URL in a file only you can read:
     ```bash
     echo 'HC_WS_TO_ACTUAL_URL=https://hc-ping.com/your-uuid-here' > ~/.job-monitoring.conf
     chmod 600 ~/.job-monitoring.conf
     ```
     This file is in your home folder, not `/etc/job-monitoring.conf`, because this job runs from **your** crontab, not root's.
   - The `exit=N` line at the end of each run is what the [login status message](/Pi-Guide/Job-Monitoring.md#login-status-message) reads. To show this job there, add it to the `JOBS` list with the path to today's log, e.g. `"ws-to-actual|/home/pi/ws_to_actual_$(date +%F).log|ws_to_actual.py|1"`.

1. Open your crontab:
   ```bash
   crontab -e
   ```
1. Add jobs to run every 12 hours and rotate logs:
   ```
   0 */12 * * * /home/pi/run_ws_to_actual.sh >> /home/pi/ws_to_actual_$(date +\%Y-\%m-\%d).log 2>&1
   10 0 * * * find /home/pi -maxdepth 1 -name "ws_to_actual_*.log" ! -name "ws_to_actual_$(date +\%Y-\%m-\%d).log" -delete
   ```
   - `-maxdepth 1` keeps the cleanup to your home folder itself. Without it, `find` also searches every folder below it, including any backups you keep there.
   - (Optional) Test it every minute first:
     ```
     * * * * * /home/pi/run_ws_to_actual.sh >> /home/pi/ws_to_actual.log 2>&1
     ```
1. Check if it ran successfully:
   ```bash
   tail -n 20 /home/pi/ws_to_actual_$(date +%F).log
   ```
   It should end with `INFO Done.` followed by `exit=0`.
1. View system cron logs if needed:
   ```bash
   journalctl -u cron --since today
   ```

## Troubleshooting

**pip not found:**

Run:
```bash
sudo apt install python3-pip -y
```

**Actual password prompt appearing in cron:**

Ensure `ACTUAL_PASSWORD` is exported in `/home/pi/ws_to_actual_env.sh` and that the wrapper script sources it as shown in [Automation](#automation).

**Wealthsimple login issues:**

Cron doesn't read `~/.bashrc`, so the wrapper sources `/home/pi/ws_to_actual_env.sh` itself. Ensure the following are set in that file:

```bash
export WS_USERNAME='your_ws_email'
export WS_PASSWORD='your_ws_pass'
export WS_TOTP_SECRET='your_ws_totp_secret'
```

If Wealthsimple still emails you about a new device even though the IP and user agent match, that means the saved session is being recreated (i.e. a full login is happening) instead of being refreshed. Check the log for which path was taken:

```bash
tail -50 ~/ws_to_actual_*.log
```

- `Found existing Wealthsimple session in keyring.` followed by `Logged in to Wealthsimple and saved session.` (no error in between) means the cached session loaded fine but the library's internal refresh silently produced a new session anyway — expected occasionally, not every run.
- `Saved WS session invalid or expired (refresh attempt failed): ...` means `from_token()` tried to refresh the access token and that path failed, forcing a brand-new interactive-style login (this is what triggers the device email).

**Known bug in `ws-api` ≤0.33.0 (fixed in 0.35.0):** if you're hitting this, upgrade:

```bash
source ~/actual_env/bin/activate
pip install --upgrade "ws-api>=0.38.1"
```

Root cause: `WealthsimpleAPI.check_oauth_token()` only checks for a top-level `message` key to detect "Not Authorized" and decide whether to attempt a refresh. Wealthsimple actually nests the error inside `errors[0].message` (e.g. `{'errors': [{'message': 'Not Authorized.', ...}]}`), so the check never matches — the refresh is never attempted, and the code falls straight through to a full login every time the ~30 min access token expires, regardless of cron interval. There's no separate "device registration" setting to force; the device email is tied to the login event itself, so the fix is making refresh actually work.

Clear saved session:
```bash
python -m keyring delete ws_to_actual.ws.session
rm -f ~/.ws_to_actual_session.json
```
Then rerun the script manually to reauthenticate.

**`Wealthsimple asked for a 2FA code (attempt N/3)`:**

The TOTP code was rejected, usually because `WS_TOTP_SECRET` is wrong or the Pi's clock is off (check `timedatectl`). The script tries 3 times, 30 seconds apart, then exits with an error instead of retrying forever, which could get your Wealthsimple account locked.

**A new Actual rule isn't applied to older transactions:**

The script runs your rules on **newly imported** transactions only, the way Actual's own import does, so a later sync never overwrites a category or payee you changed by hand. To apply a new rule to transactions that were already imported, do it from Actual's rule editor, which can apply a rule to the existing transactions it matches.

Each run's log shows `Committed N new and N updated transactions (N already up to date).`, or `All N transactions already up to date; nothing to commit.` when nothing changed.

**A rule doesn't catch the same merchant every time:**

Card payees usually include a store number or reference, e.g. `Credit card purchase: Dairy Queen #27344`, so a rule on **Payee is** only ever matches one location. Match on the raw description instead: in Actual's rule editor, use **Imported payee** → **contains** → `dairy queen`. The match ignores upper/lower case. One rule per category, set to match **any** of its conditions, keeps the list short, e.g. a Food rule matching `subway`, `tim horton` or `doordash`.

The script saves that raw description on every transaction it imports (`imported_payee=` in `create_transaction`). Versions of this script before that line was added didn't, so transactions they imported have no imported payee, and an **Imported payee** rule won't match them, even when applied from the rule editor.

Keep rules from overlapping: the script applies them in the order they're stored, not by Actual's own ranking, so if two rules match one transaction, which category wins isn't obvious. For example, `uber` would match both `Uber Canada/Ubertrip` and `Uber Canada/Ubereats`; use `ubertrip` and `ubereats` instead.

**An Actual schedule always shows "missed":**

The script runs your rules on each new transaction, including the rule that links a transaction to its schedule, so a synced payment marks its schedule paid as long as it matches all of the schedule's conditions. When a schedule never gets marked paid, check:

- **The amount's sign.** Money coming in is positive and money going out is negative. A paycheque schedule with an amount range of `-3,500` to `-2,500` can never match a `+3,000` deposit.
- **How close the amount is.** "Approximately" allows 7.5% either way. For bills that vary (utilities, or a plan whose promotional price ends), use **is between** with a range wide enough for the real amounts.
- **How close the date is.** The date only matches within 2 days of the scheduled one. Card charges can post a few days late, so set a schedule's date to the middle of the days its payments usually land on.
- **The payee and account.** Both must be the exact payee and account the sync creates, e.g. `Payroll: YOUR EMPLOYER` on the Cash account.

**Script not running from cron:**

Make sure cron is using your user's environment, not root's:
```bash
crontab -l
```
Avoid `sudo crontab -e` unless necessary.

## Python Script

Here is the complete `ws_to_actual.py` script. Copy this into `~/ws_to_actual.py`:

```python
#!/usr/bin/env python3
"""
ws_to_actual.py
Fetch Wealthsimple activities and import into Actual (via actualpy).

Dependencies:
    pip install "ws-api>=0.38.1" actualpy keyring python-dateutil pyotp

Notes:
 - This uses the unofficial ws-api Python library to access Wealthsimple.
 - Requires ws-api>=0.38.1 — older versions have a session-refresh bug (see repo docs' Troubleshooting section).
 - The first run will prompt for Wealthsimple credentials (and possibly OTP).
 - Credentials/session tokens are stored using the system keyring.
 - Configure ACTUAL_BASE_URL and ACTUAL_PASSWORD (environment vars or edit below).
"""

import os
import sys
import json
import decimal
import getpass
import logging
import time
import pyotp
from datetime import date, timedelta
from dateutil import parser as dateparser
from typing import List, Dict

# Wealthsimple client
from ws_api import (
    WealthsimpleAPI,
    OTPRequiredException,
    LoginFailedException,
    WSAPISession,
)

# Actual (actualpy)
from actual import Actual
from actual.queries import create_transaction, get_or_create_account, match_transaction

# Store sessions securely
import keyring

# ---------- CONFIG ----------
KEYRING_SERVICE = "ws_to_actual"
WS_KEYRING_SESSION_KEY = f"{KEYRING_SERVICE}.ws.session"
SESSION_CACHE_FILE = os.path.expanduser("~/.ws_to_actual_session.json")
# You can set these environment variables instead of editing here:
ACTUAL_BASE_URL = os.environ.get("ACTUAL_BASE_URL", "http://localhost:5006")
ACTUAL_PASSWORD = os.environ.get(
    "ACTUAL_PASSWORD", None
)  # recommended to be set in env
BUDGET_FILE_NAME = os.environ.get("ACTUAL_BUDGET_FILE", "Wealthsimple Import")
# Optional: only import transactions after this date (YYYY-MM-DD) to avoid importing the whole history
IMPORT_AFTER = os.environ.get("IMPORT_AFTER", None)  # e.g. "2025-01-01" or None
# Optional: restrict which accounts to import
ALLOWED_ACCOUNTS = {
    "Cash",
    "Credit card",
}  # can also override via environment if desired
# -----------------------------

logging.basicConfig(level=logging.INFO, format="%(asctime)s %(levelname)s %(message)s")
log = logging.getLogger("ws_to_actual")


def load_or_login_ws():
    """
    Load saved WS session from keyring, or prompt for login (email+password+otp).
    Returns a WealthsimpleAPI instance (authenticated).
    """
    # Optionally set a custom user-agent (ws-api example uses this)
    WealthsimpleAPI.set_user_agent("Mozilla/5.0 (RaspberryPi) ws-to-actual/1.0")

    # Try to load session JSON from keyring or local cache.
    # Accept old session records saved under the legacy ``session`` key too.
    def load_cached_session():
        for key in (WS_KEYRING_SESSION_KEY, "session"):
            try:
                session = keyring.get_password(KEYRING_SERVICE, key)
                if session:
                    return session
            except Exception:
                pass
        try:
            with open(SESSION_CACHE_FILE, "r", encoding="utf-8") as f:
                data = f.read().strip()
                if data:
                    return data
        except FileNotFoundError:
            pass
        except Exception:
            log.warning("Could not read local session cache %s", SESSION_CACHE_FILE)
        return None

    session_json = load_cached_session()
    username = None
    interactive = sys.stdin.isatty()

    # persist function: save session JSON to keyring, local cache
    def persist_session(sess_obj, uname):
        try:
            keyring.set_password(KEYRING_SERVICE, WS_KEYRING_SESSION_KEY, sess_obj)
            keyring.set_password(KEYRING_SERVICE, "session", sess_obj)
        except Exception:
            log.warning("Could not write session to keyring")
        try:
            # The session holds live access/refresh tokens: create the file
            # readable by this user only (a plain open() would create it 644).
            fd = os.open(SESSION_CACHE_FILE, os.O_WRONLY | os.O_CREAT | os.O_TRUNC, 0o600)
            with os.fdopen(fd, "w", encoding="utf-8") as f:
                os.fchmod(f.fileno(), 0o600)  # also tightens a file left over from older versions
                f.write(sess_obj)
        except Exception as e:
            log.warning("Could not write local session cache %s: %s", SESSION_CACHE_FILE, e)

    if session_json:
        try:
            sess = WSAPISession.from_json(session_json)
            log.info("Found existing Wealthsimple session in keyring.")
            ws = WealthsimpleAPI.from_token(
                sess,
                persist_session,
                username,
            )
            return ws
        except Exception as e:
            # This is almost always a failed *refresh* attempt (the access
            # token is only valid ~30 min, so it's expired by every cron
            # run; from_token() tries to use the refresh_token internally
            # before raising here). Logging the type/args helps tell an
            # expired refresh_token apart from other auth failures.
            log.warning(
                "Saved WS session invalid or expired (refresh attempt failed): %s: %s",
                type(e).__name__,
                e,
            )

    # No valid session: interactive login
    print(
        "Wealthsimple login required. (This will be saved to your OS keyring securely.)"
    )
    username = os.getenv("WS_USERNAME")
    password = os.getenv("WS_PASSWORD")
    secret = os.getenv("WS_TOTP_SECRET")

    # Optional fallback if env vars aren't set
    if not username or not password:
        if not interactive:
            raise RuntimeError(
                "Wealthsimple credentials are required when running from cron. "
                "Set WS_USERNAME and WS_PASSWORD in the environment."
            )
        if not username:
            username = input("Wealthsimple username (email): ").strip()
        if not password:
            password = getpass.getpass("Wealthsimple password: ")

    if not secret and not interactive:
        raise RuntimeError(
            "Wealthsimple TOTP secret is required when running from cron. "
            "Set WS_TOTP_SECRET in the environment."
        )

    otp_answer = None
    # Cap retries: with WS_TOTP_SECRET set, a rejected code used to loop forever,
    # hammering Wealthsimple's login endpoint from cron.
    max_attempts = 3

    for attempt in range(1, max_attempts + 1):
        try:
            if secret:
                otp_answer = pyotp.TOTP(secret).now()
            elif not interactive:
                raise RuntimeError(
                    "WS_TOTP_SECRET is required in cron when 2FA is enabled."
                )

            WealthsimpleAPI.login(
                username,
                password,
                otp_answer,
                persist_session,
            )
            stored = load_cached_session()
            if not stored:
                raise RuntimeError("Session wasn't saved to keyring or local cache")
            sess = WSAPISession.from_json(stored)
            ws = WealthsimpleAPI.from_token(sess, persist_session, username)
            log.info("Logged in to Wealthsimple and saved session.")
            return ws
        except OTPRequiredException:
            log.warning("Wealthsimple asked for a 2FA code (attempt %d/%d).", attempt, max_attempts)
            if secret:
                # The TOTP code was rejected; wait for the next 30 s window before retrying.
                time.sleep(30)
            else:
                otp_answer = input("2FA/TOTP code: ").strip()
        except LoginFailedException:
            log.error("Login failed.")
            if os.getenv("WS_USERNAME") and os.getenv("WS_PASSWORD"):
                raise
            username = input("Wealthsimple username (email): ").strip()
            password = getpass.getpass("Wealthsimple password: ")
            otp_answer = None
        except Exception as e:
            log.exception("Unexpected error during Wealthsimple login: %s", e)
            raise

    raise RuntimeError(f"Wealthsimple login failed after {max_attempts} attempts")


def is_interest_activity(act: Dict) -> bool:
    """
    Detect Wealthsimple interest / yield postings.
    """
    # WS sometimes provides explicit types
    activity_type = (act.get("type") or act.get("activityType") or "").upper()
    if activity_type == "INTEREST":
        return True

    # Description-based fallback
    desc = (act.get("description") or "").lower()
    if "interest" in desc:
        return True

    return False


def is_pending_activity(act: Dict) -> bool:
    """
    Return True if the Wealthsimple activity represents a pending transaction.
    Handles multiple possible WS fields defensively.
    """
    # Interest is NEVER pending
    if is_interest_activity(act):
        return False

    # Explicit boolean flags
    if act.get("isPending") is True:
        return True
    if act.get("pending") is True:
        return True

    # Status/state fields
    status = (act.get("status") or act.get("state") or "").upper()
    if status == "PENDING":
        return True

    # Some transactions are pending if they have no settlement date
    # but DO have an occurredAt date
    if act.get("occurredAt") and not act.get("settledAt"):
        # Many WS pending card transactions look like this
        if status in ("", "AUTHORIZED"):
            return True

    return False

def is_reversed_activity(act: Dict) -> bool:
    """
    Return True if the Wealthsimple activity represents a reversed,
    cancelled, or voided transaction.
    """
    # Explicit boolean flags
    if act.get("isReversal") is True:
        return True
    if act.get("isReversed") is True:
        return True
    if act.get("reversal") is True:
        return True

    # Linked reversal references
    if act.get("reversedTransactionId"):
        return True
    if act.get("originalTransactionId"):
        return True

    # Status / state based detection
    status = (act.get("status") or act.get("state") or "").upper()
    if status in {
        "REVERSED",
        "CANCELLED",
        "VOIDED",
        "FAILED",
        "DECLINED",
    }:
        return True

    # Description-based fallback (last resort)
    desc = (act.get("description") or "").lower()
    if any(word in desc for word in ("reversal", "reversed", "void", "cancelled")):
        return True

    return False


def fetch_wealthsimple_activities(ws, import_after: date | None = None) -> List[Dict]:
    """
    Fetch activities (transactions) only for ALLOWED_ACCOUNTS.
    Returns a list of normalized transaction dicts.
    """
    accounts = ws.get_accounts()
    result = []

    for acc in accounts:
        acc_name = acc.get("description") or acc.get("number") or f"WS-{acc['id']}"

        # Only process allowed accounts
        if acc_name not in ALLOWED_ACCOUNTS:
            log.info("Skipping account not in allowed list: %s", acc_name)
            continue

        acc_id = acc["id"]
        log.info("Fetching activities for account: %s", acc_name)

        acts = ws.get_activities(acc_id) or []
        for act in reversed(acts):  # oldest → newest
            try:
                # Skip pending transactions
                if is_pending_activity(act):
                    log.info(
                        "Skipping pending transaction: %s (%s)",
                        act.get("description"),
                        act.get("canonicalId") or act.get("id"),
                    )
                    continue

                # Skip reversed / cancelled transactions
                if is_reversed_activity(act):
                    log.info(
                        "Skipping reversed transaction: %s (%s)",
                        act.get("description"),
                        act.get("canonicalId") or act.get("id"),
                    )
                    continue

                occurred = act.get("occurredAt")
                if not occurred:
                    continue
                d = dateparser.parse(occurred).date()
                if import_after and d < import_after:
                    continue

                amount = decimal.Decimal(str(act.get("amount")))
                if act.get("amountSign") == "negative" and amount > 0:
                    amount = -abs(amount)
                elif act.get("amountSign") == "positive" and amount < 0:
                    amount = abs(amount)

                payee = act.get("description") or act.get("type") or "Wealthsimple"
                if acc_name.lower() == "cash":
                    if act.get("type") == "DEPOSIT":
                        if act.get("aftOriginatorName"):
                            payee = f"Payroll: {act['aftOriginatorName']}"
                        elif "direct deposit" in (act.get("description") or "").lower():
                            payee = act["description"]
                        else:
                            payee = act.get("description", "Cash Deposit")
                    elif act.get("type") == "WITHDRAWAL":
                        if act.get("eTransferName"):
                            payee = f"E-Transfer: {act['eTransferName']}"
                        elif act.get("eTransferEmail"):
                            payee = f"E-Transfer: {act['eTransferEmail']}"
                        else:
                            payee = act.get("description", "Cash Withdrawal")
                    elif act.get("type") == "SPEND":
                        if act.get("spendMerchant"):
                            payee = act["spendMerchant"]
                        else:
                            payee = act.get("description", "Card Spend")

                notes = f"WS type={act.get('type')} subtype={act.get('subType')} id={act.get('canonicalId')}"
                tx = {
                    "date": d,
                    "amount": amount,
                    "payee": payee.strip(),
                    "notes": notes,
                    "account_name": acc_name,
                    "canonical_id": act.get("canonicalId")
                    or act.get("id")
                    or f"ws-{acc_id}-{occurred}",
                }
                result.append(tx)

            except Exception as e:
                log.exception("Failed to process activity: %s", e)

    log.info("Fetched %d activities from allowed Wealthsimple accounts.", len(result))
    return result


def import_into_actual(
    transactions: List[Dict],
    actual_base_url: str,
    actual_password: str,
    budget_file_name: str,
):
    """
    Use actualpy to import the list of transactions into Actual.
    Transactions must include date, account_name, payee, notes, amount.
    Automatically creates the budget file on the Actual server if it does not exist.
    """
    from actual.exceptions import UnknownFileId

    if not actual_password:
        raise RuntimeError(
            "Actual password is required (set ACTUAL_PASSWORD env or provide it interactively)."
        )

    # Ensure all amounts are Decimal
    for t in transactions:
        if not isinstance(t["amount"], decimal.Decimal):
            t["amount"] = decimal.Decimal(str(t["amount"]))

    # Try to open the specified budget file; create it if missing
    try:
        actual = Actual(
            base_url=actual_base_url, password=actual_password, file=budget_file_name
        )
    except UnknownFileId:
        log.info(
            "Budget file '%s' not found on Actual server. Creating a new one...",
            budget_file_name,
        )
        temp_actual = Actual(base_url=actual_base_url, password=actual_password)
        temp_actual.create_budget(budget_file_name)
        temp_actual.upload_budget()

        # Give Actual a moment to register the new budget file
        time.sleep(2)

        # Now reopen with the new file
        actual = Actual(
            base_url=actual_base_url, password=actual_password, file=budget_file_name
        )

    # Entering the context downloads the budget (and raises if that fails).
    with actual:
        log.info("Using budget file: %s", budget_file_name)

        matched = []  # every reconciled transaction, so duplicates in one run pair up correctly
        new_transactions = []
        changed = 0
        for tx in transactions:
            account = get_or_create_account(actual.session, tx["account_name"])

            # Same logic as actualpy's reconcile_transaction(), unrolled so we know whether
            # each transaction is new. Matches by imported_id (financial_id) first, then
            # fuzzily by amount within +/-7 days.
            t = match_transaction(
                actual.session,
                tx["date"],
                account,
                tx["payee"],
                tx["amount"],
                tx["canonical_id"],
                matched,
            )
            if t is None:
                t = create_transaction(
                    actual.session,
                    tx["date"],
                    account,
                    tx["payee"],
                    tx["notes"],
                    None,  # category (None = uncategorized)
                    tx["amount"],
                    # Stored as financial_id, so later runs match this transaction exactly.
                    imported_id=tx["canonical_id"],
                    cleared=False,
                    # Raw description, which rules can match with "Imported payee contains".
                    imported_payee=tx["payee"],
                )
                new_transactions.append(t)
                print(f"Added transaction: {tx['date']} {tx['payee']} {tx['amount']}")
            else:
                t.notes = tx["notes"]
                t.set_date(tx["date"])
                if t.changed():
                    changed += 1
                    print(f"Updated transaction: {tx['date']} {tx['payee']} {tx['amount']}")
            matched.append(t)

        # Rules run on new transactions only, like Actual's own import, so a re-run never
        # overwrites categories/payees edited by hand on transactions imported earlier.
        if new_transactions:
            actual.run_rules(transactions=new_transactions)
        if new_transactions or changed:
            actual.commit()
            log.info(
                "Committed %d new and %d updated transactions (%d already up to date).",
                len(new_transactions),
                changed,
                len(matched) - len(new_transactions) - changed,
            )
        else:
            log.info("All %d transactions already up to date; nothing to commit.", len(matched))


def main():
    LOOKBACK_DAYS = 60

    if IMPORT_AFTER:
        import_after_date = dateparser.parse(IMPORT_AFTER).date()
        log.info("Importing Wealthsimple transactions from %s onward (IMPORT_AFTER)", import_after_date)
    else:
        import_after_date = date.today() - timedelta(days=LOOKBACK_DAYS)
        log.info("Importing Wealthsimple transactions from %s onward", import_after_date)

    # Wealthsimple login & fetch (only from allowed accounts)
    ws = load_or_login_ws()
    txs = fetch_wealthsimple_activities(ws, import_after=import_after_date)

    if not txs:
        log.info("No transactions to import.")
        return

    # Confirm Actual config / password
    if not ACTUAL_PASSWORD and not sys.stdin.isatty():
        raise RuntimeError(
            "ACTUAL_PASSWORD is required when running from cron. "
            "Set ACTUAL_PASSWORD in the environment."
        )
    actual_password = ACTUAL_PASSWORD or getpass.getpass(
        f"Password for Actual at {ACTUAL_BASE_URL}: "
    )

    # Import into Actual
    import_into_actual(txs, ACTUAL_BASE_URL, actual_password, BUDGET_FILE_NAME)
    log.info("Done.")


if __name__ == "__main__":
    try:
        main()
    except KeyboardInterrupt:
        print("\nInterrupted by user")
        sys.exit(1)
    except Exception as e:
        log.exception("Fatal error: %s", e)
        sys.exit(2)
```

## Sources

- https://github.com/gboudreau/ws-api-python
- https://github.com/actualbudget/actual
- https://docs.python.org/3/library/venv.html
- https://man7.org/linux/man-pages/man5/crontab.5.html
