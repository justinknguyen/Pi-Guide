# SSH Hardening

Secure your Pi's SSH access with key-based login, a firewall, and fail2ban. Strongly recommended before exposing anything to the internet (e.g., [NGINX](/Pi-Guide/NGINX.md) or [PiVPN](/Pi-Guide/PiVPN.md)).

## Table of Contents

- [Key-Based Login](#key-based-login)
- [Adding More Devices](#adding-more-devices)
- [Disable Password Login](#disable-password-login)
- [Firewall (UFW)](#firewall-ufw)
  - [What UFW Blocks](#what-ufw-blocks)
  - [Restricting a Docker Port to Your LAN](#restricting-a-docker-port-to-your-lan)
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
1. Allow SSH first, **from your own network only**. Replace `192.168.50.0/24` with your LAN (your router's IP with a `0` last digit, plus `/24`):
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 22 proto tcp comment 'SSH (LAN)'
   ```
   - Why not just `sudo ufw allow ssh`? That allows SSH from *anywhere*, over IPv6 too. Most Pis have a public IPv6 address, and if your router doesn't block unsolicited inbound IPv6 (many don't), SSH is then reachable from the whole internet. Check whether yours has one with `ip -6 addr show scope global`.
   - If you SSH in over IPv6 on your LAN, also allow your IPv6 prefix (the first four groups of that address, then `::/64`):
     ```bash
     sudo ufw allow from [YOURIPV6PREFIX]::/64 to any port 22 proto tcp comment 'SSH (LAN IPv6)'
     ```
   - Before continuing, check which address your current session came from with `echo $SSH_CONNECTION` (the first field). If it isn't covered by the rules above, enabling the firewall will cut you off.
   - Reaching the Pi from outside your home? Use [Tailscale](/Pi-Guide/Tailscale.md) (rule below) or [PiVPN](/Pi-Guide/PiVPN.md) rather than opening SSH to the internet.
1. Allow the ports for the services you run on this Pi. [What UFW Blocks](#what-ufw-blocks) below lists the services from these guides that need a rule. Services in ordinary Docker containers mostly don't, since their published ports bypass UFW. A few common ones:
   ```bash
   sudo ufw allow 80,443/tcp  # NGINX / Pi-hole web interface
   sudo ufw allow 51820/udp   # PiVPN (WireGuard)
   ```
   - The same rules are harmless for Dockerized apps like NGINX Proxy Manager, wg-easy (51821) or Vaultwarden (8200), but they aren't what keeps those reachable.
1. If this Pi shares files with [Samba](/Pi-Guide/NAS.md), allow its ports from your network. Samba runs directly on the Pi rather than in Docker, so the firewall blocks it like anything else:
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 445,139 proto tcp comment 'Samba (LAN)'
   sudo ufw allow from 192.168.50.0/24 to any port 137,138 proto udp comment 'Samba NetBIOS (LAN)'
   sudo ufw allow from [YOURIPV6PREFIX]::/64 to any port 445,139 proto tcp comment 'Samba (LAN IPv6)'
   ```
   Ports 137/138 (older network browsing) are IPv4-only, so they have no IPv6 rule.
1. If this Pi runs [Pi-hole](/Pi-Guide/Pi-hole.md), allow DNS (port 53) **only from your own network**, not from anywhere. Replace `192.168.50.0/24` with your LAN (your router's IP with a `0` last digit, plus `/24`):
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 53 comment 'Pi-hole DNS (LAN)'
   ```
   - Why not just `sudo ufw allow 53`? That rule also applies to IPv6, where the Pi usually has a public address. Many routers block unsolicited inbound IPv6, but not all — and if yours doesn't, your Pi-hole becomes an open DNS server for the whole internet, which gets abused for DDoS attacks.
   - If you use IPv6 DNS on your LAN, also allow your IPv6 prefix. Find it with `ip -6 addr show scope global` (the first four groups of the address, then `::/64`):
     ```bash
     sudo ufw allow from [YOURIPV6PREFIX]::/64 to any port 53 comment 'Pi-hole DNS (LAN IPv6)'
     ```
     Many ISPs change this prefix from time to time. If IPv6 devices suddenly lose ad-blocking while IPv4 keeps working, this rule is out of date — update it, and every other rule that uses the old prefix (SSH, Samba), with the new one.
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
   - `ufw enable` asks a y/n question. If you're running it somewhere that can't answer the prompt (a script, or a tool that passes commands through), use `sudo ufw --force enable` instead.
1. Keep your current SSH session open, and from **another device** confirm that a new SSH connection still works and that each service you allowed still opens.

Things to know about UFW:

- **Docker publishes container ports by writing its own iptables rules, which bypass UFW** — a `-p 8080:80` container is reachable even if UFW doesn't allow it. UFW still protects everything running directly on the Pi; just don't assume it covers Docker containers. The simplest way to limit a container is to publish it on a specific address, e.g. `-p 127.0.0.1:8080:80` (only reachable from the Pi itself, for use behind a reverse proxy like [NGINX](/Pi-Guide/NGINX.md)). To keep a port reachable from your LAN but nothing else, see [Restricting a Docker Port to Your LAN](#restricting-a-docker-port-to-your-lan).
- **Containers using `network_mode: host` (or `--network host`) are *not* bypassed.** They don't get Docker's port rules. They listen straight on the Pi like a normal program, so UFW blocks them unless you allow their port. See [What UFW Blocks](#what-ufw-blocks) for the list.
- **The reverse *does* go through UFW: a container reaching the Pi's own IP.** If a container calls `http://[PIIPADDRESS]:[PORT]` (e.g. [Homepage](/Pi-Guide/Homepage.md) or [Uptime Kuma](/Pi-Guide/Uptime-Kuma.md) checking a service on the same Pi), that traffic comes from Docker's internal network, which your LAN rule doesn't match. So it's silently blocked, **even when the port belongs to another Docker container**. Allow only the port it needs:
  ```bash
  sudo ufw allow from 172.16.0.0/12 to any port [PORT] proto tcp
  ```
  Avoid allowing all of `172.16.0.0/12` with no port — that would open SSH and every other service to every container.
  - **`172.16.0.0/12` isn't the only range Docker uses.** Docker gives each compose project its own network, and after working through the `172.x` ranges it moves on to `192.168.x.0/20` networks. It doesn't reuse a freed range straight away, so this can happen with only a handful of stacks. Check every network's range:
    ```bash
    docker network ls -q | xargs docker network inspect --format '{{.Name}} {{range .IPAM.Config}}{{.Subnet}}{{end}}'
    ```
    Any network outside `172.16.0.0/12` needs its own rule with that exact subnet. Don't widen it to `192.168.0.0/16`, which would also cover your LAN.
  - **`docker compose down` deletes the project's network**, and the next `up` can create it on a different range, outside your rule. That silently breaks things again. Use `docker compose stop`/`start` when you only want the containers stopped, and re-run the check above after any `down`.
  - Blocked attempts show up in `sudo journalctl -k | grep 'UFW BLOCK'` with a `SRC=172.…` or `SRC=192.168.…` address. That only lists a flow after it has actually been tried, though. A dashboard widget or monitor that hasn't polled since you enabled UFW is already broken without showing up yet. Check the containers that talk to the Pi's IP rather than waiting for the log.
- **`systemctl status ufw` says `inactive (dead)` even when the firewall is working.** It's a one-shot service that exits after loading the rules. Use `sudo ufw status` to check.
- **Testing a port from the Pi itself proves nothing.** A connection from the Pi to its own IP never passes through the firewall, so it always succeeds. Test from another device on your network (e.g., open the page on your phone).
- To see what's being blocked, `sudo ufw logging low`, then watch `sudo journalctl -k -f | grep 'UFW BLOCK'`.

### What UFW Blocks

A service stops working once UFW is enabled if it falls into one of the groups below and has no allow rule. Ports published by an ordinary Docker container (`-p` / `ports:`) aren't in the list, because those bypass UFW.

**Installed directly on the Pi, or a container with host networking.** Allow these from your LAN (`sudo ufw allow from 192.168.50.0/24 to any port [PORT] proto [tcp|udp]`). UFW's defaults already let in mDNS (5353) and UPnP/SSDP (1900) discovery, so those don't need rules:

| Guide | Ports |
| --- | --- |
| [Jellyfin](/Pi-Guide/Jellyfin.md) | 8096/tcp, plus 7359/udp so apps can find the server automatically |
| [Home Assistant](/Pi-Guide/Home-Assistant.md) | 8123/tcp |
| [Scrypted](/Pi-Guide/Scrypted.md) | 10443/tcp, plus the HomeKit ports — see that guide |
| [diyHue](/Pi-Guide/diyHue.md) | 80,443/tcp, plus 2100/udp for Hue Sync / entertainment areas |
| [Mosquitto](/Pi-Guide/Mosquitto.md) | 1883/tcp |
| [Syncthing](/Pi-Guide/Syncthing.md) | 8384/tcp (web UI), 22000/tcp+udp (syncing), 21027/udp (finding devices on your LAN) |
| [Hyperion](/Pi-Guide/Hyperion.md) / [HyperHDR](/Pi-Guide/HyperHDR.md) | 8090/tcp |
| [XRDP](/Pi-Guide/XRDP.md) | 3389/tcp |
| [Grafana](/Pi-Guide/Grafana.md)'s node-exporter | 9100/tcp — only if Prometheus runs on a *different* machine. On the same Pi, see the next table |
| [keepalived](/Pi-Guide/keepalived.md) | VRRP from the other Pi — see that guide. Without it, both Pis take over the shared IP at once |

**Containers that call the Pi's own IP.** These need a rule from Docker's networks (see [the note above](#firewall-ufw) about which ranges), not from your LAN:

| Guide | Calls |
| --- | --- |
| [Homepage](/Pi-Guide/Homepage.md), [Uptime Kuma](/Pi-Guide/Uptime-Kuma.md) | every service they link to, monitor or show a widget for |
| [Grafana](/Pi-Guide/Grafana.md) (Prometheus) | 9090 (itself) and 9100 (node-exporter), because `prometheus.yml` targets the Pi's IP |
| [Zigbee2MQTT](/Pi-Guide/Zigbee2MQTT.md) | 1883 (Mosquitto) |
| [Seerr](/Pi-Guide/Arr-Stack.md#seerr-optional) | 8096 (Jellyfin) |
| [nebula-sync](/Pi-Guide/Nebula-Sync.md) | the Pi-hole web port, when one of the Pi-holes is the Pi it runs on |

### Restricting a Docker Port to Your LAN

Some containers have endpoints that don't ask for a password: a media server's playlist URL, a stats page, an admin API meant for the LAN. UFW can't restrict their published port. Publishing on `127.0.0.1` would cut off the rest of your network too. Docker checks a chain called `DOCKER-USER` before its own rules, so you can put the restriction there.

1. Create the script:
   ```bash
   sudo nano /usr/local/sbin/docker-user-rules.sh
   ```
1. Paste the following in, replacing `LAN` with your network and `PORTS` with the ports to restrict:
   ```bash
   #!/bin/bash
   # Restrict published Docker ports to the LAN and to Docker's own networks.
   set -u
   LAN=192.168.50.0/24
   DOCKER_NETS=(172.16.0.0/12)   # add any network outside this range (see SSH Hardening)
   PORTS=(9191)                  # the port INSIDE the container, see below

   iptables -L DOCKER-USER -n >/dev/null 2>&1 || { echo "DOCKER-USER missing - is Docker running?"; exit 1; }

   # Remove this script's old rules first, so running it twice doesn't duplicate them.
   # `|| break` matters: without it, a delete that fails loops forever, at boot.
   while n=$(iptables -L DOCKER-USER --line-numbers -n | awk '/docker-user-rules/ {print $1; exit}'); [[ -n $n ]]; do
       iptables -D DOCKER-USER "$n" || break
   done

   rc=0
   for port in "${PORTS[@]}"; do
       ok=1
       for src in "$LAN" "${DOCKER_NETS[@]}"; do
           iptables -A DOCKER-USER -p tcp --dport "$port" -s "$src" -m comment --comment "docker-user-rules" -j RETURN || ok=0
       done
       # Only add the DROP if every allow went in. Otherwise a typo in one address
       # would block the LAN or your other containers along with everyone else.
       if (( ok )) && iptables -A DOCKER-USER -p tcp --dport "$port" -m comment --comment "docker-user-rules" -j DROP; then
           echo "restricted tcp/$port"
       else
           echo "FAILED to restrict tcp/$port - check: iptables -L DOCKER-USER -n"; rc=1
       fi
   done
   # A non-zero exit shows the service as failed, instead of looking applied.
   exit "$rc"
   ```
   - `--dport` matches the port **inside the container** (the right-hand side of `-p 8080:80`), because Docker has already rewritten the destination by the time traffic reaches `DOCKER-USER`. For `-p 9191:9191` they're the same.
   - **Keep Docker's networks in the allow list.** Other containers reaching this one by the Pi's IP arrive from a Docker address, not a LAN one. A rule that only allows the LAN would cut them off.
1. Make it executable, and create a service to run it:
   ```bash
   sudo chmod 700 /usr/local/sbin/docker-user-rules.sh
   sudo nano /etc/systemd/system/docker-user-rules.service
   ```
   ```ini
   [Unit]
   Description=Apply DOCKER-USER iptables rules
   After=docker.service
   Requires=docker.service
   PartOf=docker.service

   [Service]
   Type=oneshot
   RemainAfterExit=yes
   ExecStart=/usr/local/sbin/docker-user-rules.sh

   [Install]
   WantedBy=multi-user.target
   ```
   `PartOf=docker.service` is the important line. Rules in `DOCKER-USER` don't survive a reboot or a Docker restart. Without it, updating Docker quietly reopens the port.
1. Enable it and check the rules:
   ```bash
   sudo systemctl daemon-reload
   sudo systemctl enable --now docker-user-rules
   sudo iptables -L DOCKER-USER -n --line-numbers
   ```
1. Test from a device on your LAN (it should connect), and from outside it, e.g. a phone on mobile data with WiFi off (it should time out). Testing from the Pi itself proves nothing, as above.

IPv6 needs no rule as long as Docker's IPv6 support is off, which is the default. The port is then served over IPv6 by an ordinary program on the Pi, and UFW already blocks that. If you ever enable IPv6 in Docker, add matching `ip6tables` rules.

Undo: `sudo systemctl disable --now docker-user-rules && sudo iptables -F DOCKER-USER`.

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
