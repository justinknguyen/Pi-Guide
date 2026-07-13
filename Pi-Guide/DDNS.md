# Dynamic DNS (DuckDNS)

Most home ISPs change your public IP address from time to time, which breaks anything that connects to your home from outside — [PiVPN](/Pi-Guide/PiVPN.md) profiles, [NGINX](/Pi-Guide/NGINX.md) host records, etc. Dynamic DNS gives you a hostname (e.g., `myhome.duckdns.org`) that automatically follows your public IP. DuckDNS is free.

## Table of Contents

- [Installation](#installation)
- [Testing](#testing)
- [Using Your Hostname](#using-your-hostname)
- [Alternatives](#alternatives)
- [Sources](#sources)

## Installation

1. Sign in at https://www.duckdns.org (GitHub/Google login), create a subdomain, and note the token shown at the top of the page.
1. On the Pi, create a folder and update script:
   ```bash
   mkdir ~/duckdns
   nano ~/duckdns/duck.sh
   ```
1. Paste the following in, replacing `[YOURSUBDOMAIN]` (just the name, not the full URL) and `[YOURTOKEN]`:
   ```bash
   echo url="https://www.duckdns.org/update?domains=[YOURSUBDOMAIN]&token=[YOURTOKEN]&ip=" | curl -k -o ~/duckdns/duck.log -K -
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then make it executable:
   ```bash
   chmod 700 ~/duckdns/duck.sh
   ```
1. Schedule it to run every 5 minutes:
   ```bash
   crontab -e
   ```
   ```
   */5 * * * * ~/duckdns/duck.sh >/dev/null 2>&1
   ```

## Testing

Run the script once and check the log — it should contain `OK`:

```bash
~/duckdns/duck.sh
cat ~/duckdns/duck.log
```

## Using Your Hostname

- **PiVPN**: use `[YOURSUBDOMAIN].duckdns.org` as the endpoint in your WireGuard client configs (or select the DNS-name option during PiVPN setup).
- **NGINX/HTTPS**: point a `CNAME` record on your own domain at the DuckDNS hostname, or use the DuckDNS hostname directly.

## Alternatives

- Many routers have built-in DDNS (e.g., Asus DDNS under WAN settings) — if yours does, that's one less thing running on the Pi.
- If you own a domain on Cloudflare, a `cloudflare-ddns` container does the same thing with your own domain name.

## Sources

- https://www.duckdns.org/install.jsp
