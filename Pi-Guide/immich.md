# immich

Automatically backup your photos and videos to your Raspberry Pi. No more iCloud!

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

- [Docker](/Pi-Guide/Docker.md)
- [Portainer](/Pi-Guide/Portainer.md) (recommended)
- [NAS](/Pi-Guide/NAS.md) (recommended)

## Installation

1. First create a folder for immich. For example, I have an external ssd that I have mounted on `/mnt/sda1` and a `shared` folder within it that is exposed to my network (see [NAS](/Pi-Guide/NAS.md)):
   ```
   sudo mkdir /mnt/sda1/shared/immich-app
   ```
1. Create a folder to store your photos and videos:
   ```
   sudo mkdir /mnt/sda1/shared/photos-and-videos
   ```
1. cd to the immich folder:
   ```
   cd /mnt/sda1/shared/immich-app
   ```
1. Get the docker, .env, and hwaccel files:
   ```
   sudo wget https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
   sudo wget -O .env https://github.com/immich-app/immich/releases/latest/download/example.env
   sudo wget https://github.com/immich-app/immich/releases/latest/download/hwaccel.transcoding.yml
   sudo wget https://github.com/immich-app/immich/releases/latest/download/hwaccel.ml.yml
   ```
1. Open the .env file:
   ```
   sudo nano .env
   ```
   - change the upload location to:
     ```
     /mnt/sda1/shared/photos-and-videos
     ```
   - at the bottom of the file, add the line:
     ```
     NODE_OPTIONS=--max-old-space-size=8192
     ```
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

## Sources

- https://immich.app/docs/install/docker-compose
- https://immich.app/docs/overview/quick-start
- https://github.com/immich-app/immich/issues/4530
