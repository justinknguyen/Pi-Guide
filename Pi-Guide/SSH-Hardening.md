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

1. Create a drop-in config file rather than editing the main `sshd_config` — it keeps your changes in one place and survives package updates that replace the main file:
   ```bash
   sudo nano /etc/ssh/sshd_config.d/10-hardening.conf
   ```
1. Paste the following in and save with `Ctrl+X` then `Y` then `Enter`:
   ```
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin no
   ```
1. Check the config for errors before applying it — a typo here can stop SSH from starting:
   ```bash
   sudo sshd -t
   ```
   No output means it's valid.
1. Apply it with `reload`, which keeps your current session connected (`restart` can drop it):
   ```bash
   sudo systemctl reload ssh
   ```
1. Confirm the setting that SSH is *actually* using:
   ```bash
   sudo sshd -T | grep -Ei 'passwordauthentication|permitrootlogin'
   ```
   It should show `passwordauthentication no`. If it still says `yes`, another file is overriding yours: for each setting, SSH uses the **first** value it reads, and the files in `/etc/ssh/sshd_config.d/` are read in alphabetical order before the rest of `sshd_config`. Some images (e.g. ones set up by cloud-init) ship a file there that turns password login back on. Look with `grep -r PasswordAuthentication /etc/ssh/sshd_config.d/`, and make sure yours sorts first (hence the `10-` prefix).
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
1. Allow the ports for the services you run on this Pi. This repo has grown a lot of guides, each with its own port — check the one you're using rather than assuming this list is complete. A few common ones:
   ```bash
   sudo ufw allow 80,443/tcp  # NGINX / Pi-hole web interface / NGINX Proxy Manager
   sudo ufw allow 51820/udp   # PiVPN (WireGuard)
   sudo ufw allow 51821/tcp   # wg-easy admin UI
   sudo ufw allow 8200/tcp    # Vaultwarden
   ```
1. If this Pi runs [Pi-hole](/Pi-Guide/Pi-hole.md), allow DNS (port 53) **only from your own network**, not from anywhere. Replace `192.168.50.0/24` with your LAN (your router's IP with a `0` last digit, plus `/24`):
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 53 comment 'Pi-hole DNS (LAN)'
   ```
   - Why not just `sudo ufw allow 53`? That rule also applies to IPv6, where the Pi usually has a public address. Many routers block unsolicited inbound IPv6, but not all — and if yours doesn't, your Pi-hole becomes an open DNS server for the whole internet, which gets abused for DDoS attacks.
   - If you use IPv6 DNS on your LAN, also allow your IPv6 prefix. Find it with `ip -6 addr show scope global` (the first four groups of the address, then `::/64`):
     ```bash
     sudo ufw allow from [YOURIPV6PREFIX]::/64 to any port 53 comment 'Pi-hole DNS (LAN IPv6)'
     ```
     Many ISPs change this prefix from time to time. If IPv6 devices suddenly lose ad-blocking while IPv4 keeps working, this rule is out of date — update it with the new prefix.
1. If this Pi runs [Tailscale](/Pi-Guide/Tailscale.md), allow traffic arriving over the tailnet (this also covers Pi-hole for your devices away from home):
   ```bash
   sudo ufw allow in on tailscale0 comment 'Tailscale'
   ```
   If the Pi is also a subnet router or exit node, see [Tailscale's firewall notes](/Pi-Guide/Tailscale.md#firewall-ufw) too.
1. Enable the firewall and check its status:
   ```bash
   sudo ufw enable
   sudo ufw status
   ```

Things to know about UFW:

- **Docker publishes container ports by writing its own iptables rules, which bypass UFW** — a `-p 8080:80` container is reachable even if UFW doesn't allow it. UFW still protects everything running directly on the Pi; just don't assume it covers Docker containers. The simplest way to limit a container is to publish it on a specific address, e.g. `-p 127.0.0.1:8080:80` (only reachable from the Pi itself, for use behind a reverse proxy like [NGINX](/Pi-Guide/NGINX.md)). For anything more, Docker's docs cover filtering with the `DOCKER-USER` chain (see Sources).
- **The reverse *does* go through UFW: a container reaching the Pi's own IP.** If a container calls `http://[PIIPADDRESS]:[PORT]` (e.g. [Homepage](/Pi-Guide/Homepage.md) checking a service on the same Pi), that traffic comes from Docker's internal network (`172.16.0.0/12`), which your LAN rule doesn't match — so it's silently blocked. Allow only the port it needs:
  ```bash
  sudo ufw allow from 172.16.0.0/12 to any port [PORT] proto tcp
  ```
  Avoid allowing all of `172.16.0.0/12` with no port — that would open SSH and every other service to every container.
- **`systemctl status ufw` says `inactive (dead)` even when the firewall is working.** It's a one-shot service that exits after loading the rules. Use `sudo ufw status` to check.
- **Testing a port from the Pi itself proves nothing.** A connection from the Pi to its own IP never passes through the firewall, so it always succeeds. Test from another device on your network (e.g., open the page on your phone).
- To see what's being blocked, `sudo ufw logging low`, then watch `sudo journalctl -k -f | grep 'UFW BLOCK'`.

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
- https://docs.docker.com/engine/network/packet-filtering-firewalls/
