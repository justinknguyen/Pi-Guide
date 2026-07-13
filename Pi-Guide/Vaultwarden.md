# Vaultwarden

Lightweight self-hosted Bitwarden server — a full password manager on your Pi that works with the official Bitwarden apps and browser extensions.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Enable HTTPS (Required)](#enable-https-required)
- [Configuration](#configuration)
- [Importing Passwords From iCloud or Your Browser](#importing-passwords-from-icloud-or-your-browser)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- A way to serve it over HTTPS — see [Enable HTTPS (Required)](#enable-https-required)

## Installation

Run Vaultwarden (this maps it to port `8200` to avoid clashing with other guides in this repo):

```bash
docker run -d --name vaultwarden --restart unless-stopped -v /home/pi/vw-data/:/data/ -p 8200:80 vaultwarden/server:latest
```

## Enable HTTPS (Required)

Bitwarden clients (and the web vault's crypto) refuse to work over plain HTTP, so you need HTTPS in front of Vaultwarden. Pick one:

- **NGINX reverse proxy + certbot** — follow [NGINX](/Pi-Guide/NGINX.md) (requires a domain, e.g. via [DDNS](/Pi-Guide/DDNS.md)), and proxy a subdomain to `http://127.0.0.1:8200`. The Vaultwarden wiki (under Sources) has ready-made proxy configs.
- **Tailscale** — if you only need access on your own devices, follow [Tailscale](/Pi-Guide/Tailscale.md) and serve it inside your tailnet with a valid certificate, no port forwarding or domain needed:
  ```bash
  sudo tailscale serve --bg 8200
  ```
  Your vault is then at `https://<pi-name>.<tailnet>.ts.net`.

## Configuration

Once your account is created, disable open registration so strangers can't sign up (important if it's internet-facing). Recreate the container with `SIGNUPS_ALLOWED=false`:

```bash
docker stop vaultwarden && docker rm vaultwarden
docker run -d --name vaultwarden --restart unless-stopped -e SIGNUPS_ALLOWED=false -v /home/pi/vw-data/:/data/ -p 8200:80 vaultwarden/server:latest
```

Your data is kept in `/home/pi/vw-data` — include it in your backups. Passwords are the one thing you really don't want to lose, so back this folder up off the Pi (see [Rclone](/Pi-Guide/Rclone.md)).

## Importing Passwords From iCloud or Your Browser

If your passwords currently live in iCloud Keychain or a browser, export them to CSV and import that into your vault:

1. Export:
   - **iCloud Keychain**: on a Mac, open the Passwords app (System Settings > Passwords on older macOS), then File > Export All Passwords. On Windows, recent versions of iCloud for Windows can export from the iCloud Passwords app.
   - **Chrome/Edge/Brave**: go to the browser's password settings (`edge://settings/passwords`, `brave://settings/passwords`, etc.), open the `...` menu next to saved passwords, and select Export.
1. Log in to your Vaultwarden web vault, go to Tools > Import data, and pick the matching format:
   - `Safari and macOS (csv)` for the iCloud export
   - `Chrome (csv)` for Chrome, Edge, or Brave (all Chromium-based)
1. Upload the file and import.
1. <ins>IMPORTANT:</ins> the CSV contains every password in plaintext — delete it immediately after importing and empty the trash.
1. Switch autofill over so you don't end up with two vaults drifting apart:
   - Install the Bitwarden browser extension (set the server URL to your Vaultwarden address) and disable the browser's built-in "offer to save passwords" setting.
   - On iPhone/iPad: Settings > Passwords > Password Options, enable Bitwarden and untick iCloud Keychain.

## Testing

Open your HTTPS address in a browser, create your account, then log in from the Bitwarden app/extension by setting the server URL to the same address (the "self-hosted" option on the login screen).

## Updating

```bash
docker stop vaultwarden && docker rm vaultwarden
docker pull vaultwarden/server:latest
```

Then re-run the `docker run` command from [Configuration](#configuration).

## Sources

- https://github.com/dani-garcia/vaultwarden
- https://github.com/dani-garcia/vaultwarden/wiki
- https://tailscale.com/kb/1242/tailscale-serve
