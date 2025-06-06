# immich

Automatically backup your photos and videos to your Raspberry Pi. No more iCloud!

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Updating](#updating)
- [Backup and Restore](#backup-and-restore)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [Portainer](/Pi-Guide/Portainer.md) (recommended)
- [NAS](/Pi-Guide/NAS.md) (recommended if you have an external drive)

## Installation

1. First create a folder for immich. For example, I have an external ssd that I have mounted on `/mnt/sda1` (follow [NAS](/Pi-Guide/NAS.md) to mount a drive):
   ```
   sudo mkdir /mnt/sda1/immich-app
   ```
   - you can view where your drive is mounted to by entering:
     ```
     lsblk
     ```
1. cd to the immich folder you created in Step 1:
   ```
   cd /mnt/sda1/immich-app
   ```
1. Get the docker, .env, and hwaccel files:
   ```
   sudo wget -O docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
   sudo wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
   ```
1. Open the .env file:
   ```
   sudo nano .env
   ```
   - at the bottom of the file, add the line:
     ```
     NODE_OPTIONS=--max-old-space-size=8192
     ```
1. If you're using an external drive for immich, please see the following section [Docker Containers Depending on External Drive](/Pi-Guide/NAS.md#docker-containers-depending-on-external-drive)
1. Start the containers:
   ```
   docker compose up -d
   ```

## Configuration

1. Access the Web UI:
   ```
   [PIIPADDRESS]:2283
   ```
1. Click on Getting Started and create your account and login

You're done! You can download the immich app from the app store and start backing up your photos and videos automatically.

## Updating

1. cd to the immich folder:
   ```
   cd /mnt/sda1/immich-app
   ```
1. Pull and update:
   ```
   docker compose pull && docker compose up -d
   ```

## Backup and Restore

https://immich.app/docs/administration/backup-and-restore

## Sources

- https://immich.app/docs/install/docker-compose
- https://immich.app/docs/overview/quick-start
- https://github.com/immich-app/immich/issues/4530
- https://immich.app/docs/administration/backup-and-restore/
