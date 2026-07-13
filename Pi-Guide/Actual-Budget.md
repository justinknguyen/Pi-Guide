# Actual Budget

Self-hosted budgeting app with envelope-style budgeting, bank-style reconciliation, and apps for web and mobile. This is the server that [Wealthsimple to Actual Budget Sync](/Pi-Guide/Wealthsimple-to-ActualBudget-Sync.md) imports into.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Updating](#updating)
- [Backups](#backups)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

1. Create a folder and a compose file:
   ```bash
   mkdir ~/actual
   nano ~/actual/docker-compose.yml
   ```
1. Paste the following in:
   ```yaml
   services:
     actual_server:
       image: actualbudget/actual-server:latest
       container_name: actual_server
       ports:
         - '5006:5006'
       volumes:
         - ./actual-data:/data
       restart: unless-stopped
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`. Then start it:
   ```bash
   cd ~/actual
   docker compose up -d
   ```

## Testing

Open `[PIIPADDRESS]:5006` in your address bar. On first visit you'll set a server password, then create your first budget file. Use the same address and password in the mobile/desktop apps.

## Updating

```bash
cd ~/actual
docker compose pull && docker compose up -d
```

## Backups

All data lives in `~/actual/actual-data` — include that folder in your [raspiBackup](/Pi-Guide/raspiBackup.md) or [Rclone](/Pi-Guide/Rclone.md) backups. Actual also supports exporting the budget from Settings as a zip.

## Sources

- https://actualbudget.org/docs/install/docker
