# Portainer

Provides GUI for Docker containers to easily manage.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

1. Update and Upgrade:
   ```
   sudo apt update
   sudo apt upgrade
   ```
2. Install Portainer
   ```
   sudo docker pull portainer/portainer-ce:latest
   ```
3. Run Portainer
   ```
   sudo docker run -d -p 9000:9000 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
   ```

## Testing

You can now access the WebUI by typing `[PIIPADDRESS]:9000` into your search bar. Follow the link under Sources to learn how to use Portainer.

## Updating

1. Stop Portainer
   ```
   docker stop portainer
   ```
2. Remove Portainer
   ```
   docker rm portainer
   ```
3. Repeat steps 2-3 under `Installation`.

## Sources

- https://pimylifeup.com/raspberry-pi-portainer/
