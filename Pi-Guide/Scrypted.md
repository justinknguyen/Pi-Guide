# Scrypted

Camera hub that bridges almost any IP camera into Apple HomeKit (including HomeKit Secure Video), Google Home, or Alexa — with noticeably faster stream load times than most native integrations. The self-hosted answer to camera subscriptions.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- An IP camera Scrypted supports — anything with RTSP works, plus native plugins for Unifi, Amcrest, Reolink, Hikvision, Tuya, Wyze, and more

## Installation

1. Create the data folder and run Scrypted (host networking is required for HomeKit discovery):
   ```bash
   mkdir -p ~/.scrypted/volume
   docker run -d --name scrypted --restart unless-stopped --network host -v ~/.scrypted/volume:/server/volume ghcr.io/koush/scrypted
   ```

## Configuration

1. Open `https://[PIIPADDRESS]:10443` in your address bar — note it's `https`, and your browser will warn about a self-signed certificate; proceed anyway.
1. Create your admin account.
1. Install a plugin for your camera brand (Plugins > Install), or the generic "RTSP Camera" plugin, and add your camera with its address and credentials.
1. Install the "HomeKit" plugin. Each camera gets a pairing QR code / setup code on its HomeKit settings page — scan it from the Home app on your iPhone (Add Accessory).
1. For HomeKit Secure Video recording, enable the camera's recording option in the Home app (requires an iCloud+ plan, as with any HKSV camera).

## Testing

Open the Home app and view the camera's live stream. It should load in about a second — noticeably faster than most cameras' native HomeKit implementations.

## Updating

```bash
docker stop scrypted && docker rm scrypted
docker pull ghcr.io/koush/scrypted
docker run -d --name scrypted --restart unless-stopped --network host -v ~/.scrypted/volume:/server/volume ghcr.io/koush/scrypted
```

Your settings are kept in `~/.scrypted/volume`.

## Sources

- https://github.com/koush/scrypted
- https://docs.scrypted.app/
