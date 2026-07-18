# SSH Hardening

Secure your Pi's SSH access with key-based login, a firewall, and fail2ban. Strongly recommended before exposing anything to the internet (e.g., [NGINX](/Pi-Guide/NGINX.md) or [PiVPN](/Pi-Guide/PiVPN.md)).

## Table of Contents

- [Key-Based Login](#key-based-login)
- [Adding More Devices](#adding-more-devices)
- [Disable Password Login](#disable-password-login)
- [Firewall (UFW)](#firewall-ufw)
- [Fail2ban](#fail2ban)
- [Sources](#sources)

## Key-Based Login

1. On your computer (not the Pi), generate a key pair. Press Enter through the prompts to accept the defaults:
   ```bash
   ssh-keygen -t ed25519
   ```
1. Copy the public key to the Pi (replace `pi` with your username):
   - macOS/Linux:
     ```bash
     ssh-copy-id pi@[PIIPADDRESS]
     ```
   - Windows (PowerShell):
     ```powershell
     type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh pi@[PIIPADDRESS] "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
     ```
1. Test it — this should now log you in without asking for the Pi's password:
   ```bash
   ssh pi@[PIIPADDRESS]
   ```

## Adding More Devices

`authorized_keys` can hold multiple keys, one per line — so give each device (phone, laptop, etc.) its own key pair rather than copying one private key everywhere. That way losing a device only means removing its one line, not regenerating a shared key.

1. On the new device (e.g., in Termius: Keychain → New Key), generate an ED25519 key pair.
1. Get its public key onto the Pi's `authorized_keys`:
   - **Termius:** with the key selected, use **Key export → Export to host**, pick the host, and leave the default location (`.ssh`) and filename (`authorized_keys`). This connects using the host's existing credentials (e.g., password) and appends the key for you — no manual copying needed.
   - **Manually**, from a computer that already has access. This creates `~/.ssh` if it doesn't exist yet and sets the permissions sshd requires (it silently ignores keys if permissions are too open):
     ```bash
     ssh pi@[PIIPADDRESS] "mkdir -p ~/.ssh && chmod 700 ~/.ssh && echo 'PASTE_PUBLIC_KEY_HERE' >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
     ```
1. In the SSH client on the new device, set the connection to use that key, then test it logs in.

Repeat for each additional device.

If a new key gets rejected with "Permission denied (publickey)" even though the key in `authorized_keys` looks correct, check your home directory's permissions — sshd's `StrictModes` setting (on by default) silently refuses key auth if `~` or `~/.ssh` is group- or world-writable:
```bash
ssh pi@[PIIPADDRESS] "chmod 755 ~"
```

## Disable Password Login

<ins>IMPORTANT:</ins> Only do this after confirming key-based login works, or you will lock yourself out. Keep your current SSH session open and test the login from a second terminal.

1. Open the SSH server config:
   ```bash
   sudo nano /etc/ssh/sshd_config
   ```
1. Find the `PasswordAuthentication` line, uncomment it if needed, and set it to:
   ```
   PasswordAuthentication no
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Restart SSH:
   ```bash
   sudo systemctl restart ssh
   ```
1. From a second terminal, confirm you can still log in (and that password login is refused if you try `ssh -o PubkeyAuthentication=no pi@[PIIPADDRESS]`).

## Firewall (UFW)

UFW (Uncomplicated Firewall) blocks all incoming connections except the ones you allow.

<ins>IMPORTANT:</ins> Allow SSH *before* enabling the firewall, or you will lock yourself out.

1. Install UFW:
   ```bash
   sudo apt install ufw
   ```
1. Allow SSH first:
   ```bash
   sudo ufw allow ssh
   ```
1. Allow the ports for the services you run on this Pi. Common examples from this repo:
   ```bash
   sudo ufw allow 53          # Pi-hole DNS
   sudo ufw allow 80,443/tcp  # NGINX / Pi-hole web interface
   sudo ufw allow 51820/udp   # PiVPN (WireGuard)
   ```
1. Enable the firewall and check its status:
   ```bash
   sudo ufw enable
   sudo ufw status
   ```

Note: Docker publishes container ports by writing its own iptables rules, which **bypass UFW** — a `-p 8080:80` container is reachable even if UFW doesn't allow it. UFW still protects everything running directly on the Pi; just don't assume it covers Docker containers.

## Fail2ban

fail2ban temporarily bans IP addresses that repeatedly fail to log in.

1. Install it:
   ```bash
   sudo apt install fail2ban
   ```
1. Create a local jail config:
   ```bash
   sudo nano /etc/fail2ban/jail.local
   ```
1. Paste the following in and save. (`backend = systemd` is required on Raspberry Pi OS "Bookworm"/Debian 12 and newer, which no longer write `/var/log/auth.log` by default):
   ```ini
   [sshd]
   enabled = true
   backend = systemd
   ```
1. Restart and check it's watching SSH:
   ```bash
   sudo systemctl restart fail2ban
   sudo fail2ban-client status sshd
   ```

## Sources

- https://www.raspberrypi.com/documentation/computers/configuration.html#configure-ssh-without-a-password
- https://help.ubuntu.com/community/UFW
- https://github.com/fail2ban/fail2ban
- https://github.com/moby/moby/issues/4737
