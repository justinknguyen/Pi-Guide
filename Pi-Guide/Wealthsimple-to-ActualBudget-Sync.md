# Wealthsimple to Actual Budget Sync

Automate syncing Wealthsimple transactions into ActualBudget using a Python script on your Raspberry Pi. The script connects to the Wealthsimple GraphQL API and uploads transaction data directly into ActualBudget.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Automation](#automation)
- [Troubleshooting](#troubleshooting)
- [Python Script](#python-script)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md) (if running ActualBudget in a container)

## Installation

1. Update system packages:
   ```
   sudo apt update && sudo apt upgrade
   ```
2. Install Python and pip:
   ```
   sudo apt install python3 python3-venv python3-pip -y
   ```
3. Create and activate a virtual environment:
   ```
   python3 -m venv ~/actual_env
   source ~/actual_env/bin/activate
   ```
4. Install required dependencies:
   ```
   pip install ws-api actualpy keyring python-dateutil
   ```
5. Create the `ws_to_actual.py` script in your home directory (see [Python Script](#python-script) section below for the full code):
   ```
   nano ~/ws_to_actual.py
   ```
   - Copy and paste the script from the [Python Script](#python-script) section
   - Save with `Ctrl+X`, then `Y`
   - Make it executable:
     ```
     chmod +x ~/ws_to_actual.py
     ```

## Configuration

1. Set environment variables in your `~/.bashrc` file:
   ```
   export ACTUAL_BASE_URL="http://localhost:5006"
   export ACTUAL_PASSWORD="your_actual_password"
   export ACTUAL_BUDGET_FILE="Wealthsimple Import"
   export IMPORT_AFTER="2025-01-01"
   ```
2. Apply changes:
   ```
   source ~/.bashrc
   ```
3. Run the script manually the first time:
   ```
   source ~/actual_env/bin/activate
   python ~/ws_to_actual.py
   ```
   - You'll be prompted for your Wealthsimple login and OTP (stored securely in your system keyring)
   - The first run may also ask for your Actual password if not set in the environment
4. (Optional) Limit imported accounts by editing the filter list in your script:
   ```
   ALLOWED_ACCOUNTS = ["Cash", "Credit card"]
   ```

## Testing

You can test the script manually before automating:

```
source ~/actual_env/bin/activate
python ~/ws_to_actual.py
```

Check logs:

```
tail -n 20 ~/ws_to_actual.log
```

A successful run will look like:

```
INFO Found existing Wealthsimple session in keyring.
INFO Filtered transactions from 273 → 47 (accounts: Cash, Credit card)
INFO Committed 47 transactions to Actual.
INFO Done.
```

## Automation

1. Open your crontab:
   ```
   crontab -e
   ```
2. Add a job to run every 12 hours:
   ```
   0 */12 * * * bash -c 'source /home/pi/.bashrc && /home/pi/actual_env/bin/python /home/pi/ws_to_actual.py >> /home/pi/ws_to_actual.log 2>&1'
   ```
   - (Optional) Test it every minute first:
     ```
     * * * * * bash -c 'source /home/pi/.bashrc && /home/pi/actual_env/bin/python /home/pi/ws_to_actual.py >> /home/pi/ws_to_actual.log 2>&1'
     ```
3. Check if it ran successfully:
   ```
   tail -n 20 ~/ws_to_actual.log
   ```
4. View system cron logs if needed:
   ```
   grep CRON /var/log/syslog | tail -n 20
   ```

## Troubleshooting

**pip not found:**

Run:
```
sudo apt install python3-pip -y
```

**Actual password prompt appearing in cron:**

Ensure it's exported in your `~/.bashrc` and that cron sources it as shown above.

**Wealthsimple login issues:**

Clear saved session:
```
python -m keyring delete ws_to_actual.ws.session
```
Then rerun the script manually to reauthenticate.

**Script not running from cron:**

Make sure cron is using your user's environment, not root's:
```
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
  pip install ws-api actualpy keyring python-dateutil

Notes:
 - This uses the unofficial ws-api Python library to access Wealthsimple.
 - The first run will prompt for Wealthsimple credentials (and possibly OTP).
 - Credentials/session tokens are stored using the system keyring.
 - Configure ACTUAL_BASE_URL and ACTUAL_PASSWORD (environment vars or edit below).
"""

import os
import sys
import decimal
import getpass
import logging
from datetime import datetime, date
from dateutil import parser as dateparser
from typing import List, Dict

# Wealthsimple client
from ws_api import WealthsimpleAPI, OTPRequiredException, LoginFailedException, WSAPISession

# Actual (actualpy)
from actual import Actual
from actual.queries import get_or_create_account, reconcile_transaction

# Store sessions securely
import keyring

# ---------- CONFIG ----------
KEYRING_SERVICE = "ws_to_actual"
WS_KEYRING_SESSION_KEY = f"{KEYRING_SERVICE}.ws.session"
# You can set these environment variables instead of editing here:
ACTUAL_BASE_URL = os.environ.get("ACTUAL_BASE_URL", "http://localhost:5006")
ACTUAL_PASSWORD = os.environ.get("ACTUAL_PASSWORD", None)  # recommended to be set in env
BUDGET_FILE_NAME = os.environ.get("ACTUAL_BUDGET_FILE", "Wealthsimple Import")
# Optional: only import transactions after this date (YYYY-MM-DD) to avoid importing the whole history
IMPORT_AFTER = os.environ.get("IMPORT_AFTER", None)  # e.g. "2025-01-01" or None
# Optional: restrict which accounts to import
ALLOWED_ACCOUNTS = {"Cash", "Credit card"}  # can also override via environment if desired
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

    # Try to load session JSON from keyring
    session_json = keyring.get_password(KEYRING_SERVICE, "session")
    username = None

    if session_json:
        try:
            sess = WSAPISession.from_json(session_json)
            log.info("Found existing Wealthsimple session in keyring.")
            ws = WealthsimpleAPI.from_token(sess, lambda s, u: keyring.set_password(KEYRING_SERVICE, "session", s), username)
            return ws
        except Exception as e:
            log.warning("Saved WS session invalid or expired: %s", e)

    # No valid session: interactive login
    print("Wealthsimple login required. (This will be saved to your OS keyring securely.)")
    username = input("Wealthsimple username (email): ").strip()
    password = getpass.getpass("Wealthsimple password: ")
    otp_answer = None

    # persist function: save session JSON to keyring
    def persist_session(sess_obj, uname):
        # sess_obj here is a JSON string (ws-api usage)
        keyring.set_password(KEYRING_SERVICE, "session", sess_obj)

    while True:
        try:
            WealthsimpleAPI.login(username, password, otp_answer, persist_session_fct=persist_session)
            # After login the keyring was filled by persist_session
            stored = keyring.get_password(KEYRING_SERVICE, "session")
            if not stored:
                raise RuntimeError("Session wasn't saved to keyring")
            sess = WSAPISession.from_json(stored)
            ws = WealthsimpleAPI.from_token(sess, persist_session, username)
            log.info("Logged in to Wealthsimple and saved session.")
            return ws
        except OTPRequiredException:
            otp_answer = input("2FA/TOTP code (check your app/email): ").strip()
        except LoginFailedException:
            print("Login failed. Try again.")
            username = input("Wealthsimple username (email): ").strip()
            password = getpass.getpass("Wealthsimple password: ")
            otp_answer = None
        except Exception as e:
            log.exception("Unexpected error during Wealthsimple login: %s", e)
            raise


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
                    "canonical_id": act.get("canonicalId") or act.get("id") or f"ws-{acc_id}-{occurred}",
                }
                result.append(tx)

            except Exception as e:
                log.exception("Failed to process activity: %s", e)

    log.info("Fetched %d activities from allowed Wealthsimple accounts.", len(result))
    return result


def import_into_actual(transactions: List[Dict], actual_base_url: str, actual_password: str, budget_file_name: str):
    """
    Use actualpy to import the list of transactions into Actual.
    Transactions must include date, account_name, payee, notes, amount.
    Automatically creates the budget file on the Actual server if it does not exist.
    """
    from actual.exceptions import UnknownFileId
    import time

    if not actual_password:
        raise RuntimeError("Actual password is required (set ACTUAL_PASSWORD env or provide it interactively).")

    # Ensure all amounts are Decimal
    for t in transactions:
        if not isinstance(t["amount"], decimal.Decimal):
            t["amount"] = decimal.Decimal(str(t["amount"]))

    # Try to open the specified budget file; create it if missing
    try:
        actual = Actual(base_url=actual_base_url, password=actual_password, file=budget_file_name)
    except UnknownFileId:
        log.info("Budget file '%s' not found on Actual server. Creating a new one...", budget_file_name)
        temp_actual = Actual(base_url=actual_base_url, password=actual_password)
        temp_actual.create_budget(budget_file_name)
        temp_actual.upload_budget()

        # Give Actual a moment to register the new budget file
        time.sleep(2)

        # Now reopen with the new file
        actual = Actual(base_url=actual_base_url, password=actual_password, file=budget_file_name)

    with actual:
        try:
            actual.download_budget()
            log.info("Using budget file: %s", budget_file_name)
        except Exception as e:
            log.warning("Could not download budget file: %s", e)

        added_transactions = []
        for tx in transactions:
            account = get_or_create_account(actual.session, tx["account_name"])

            t = reconcile_transaction(
                actual.session,
                tx["date"],
                account,
                tx["payee"],
                tx["notes"],
                None,  # category (None = uncategorized)
                tx["amount"],
                cleared=False,
                already_matched=added_transactions,
            )
            added_transactions.append(t)
            if t.changed():
                print(f"Added/modified transaction: {tx['date']} {tx['payee']} {tx['amount']}")

        # Commit all changes
        actual.commit()
        log.info("Committed %d transactions to Actual.", len(added_transactions))


def main():
    # Parse IMPORT_AFTER if set
    import_after_date = None
    if IMPORT_AFTER:
        import_after_date = datetime.strptime(IMPORT_AFTER, "%Y-%m-%d").date()

    # Wealthsimple login & fetch (only from allowed accounts)
    ws = load_or_login_ws()
    txs = fetch_wealthsimple_activities(ws, import_after=import_after_date)

    if not txs:
        log.info("No transactions to import.")
        return

    # Confirm Actual config / password
    actual_password = ACTUAL_PASSWORD or getpass.getpass(f"Password for Actual at {ACTUAL_BASE_URL}: ")

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

- https://github.com/ianepreston/ws-api
- https://github.com/actualbudget/actual
- https://docs.python.org/3/library/venv.html
- https://man7.org/linux/man-pages/man5/crontab.5.html
