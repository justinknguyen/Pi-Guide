# Portainer

Provides a GUI to easily manage Docker containers.

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
   ```bash
   sudo apt update
   sudo apt upgrade
   ```
1. Install and run Portainer:
   ```bash
   sudo docker pull portainer/portainer-ce:latest
   sudo docker run -d -p 8000:8000 -p 9443:9443 --name=portainer --restart=always -v /var/run/docker.sock:/var/run/docker.sock -v portainer_data:/data portainer/portainer-ce:latest
   ```

## Testing

You can now access the WebUI by typing `https://[PIIPADDRESS]:9443` into your search bar. Follow the link under Sources to learn how to use Portainer.

## Updating

1. Stop and remove Portainer
   ```bash
   docker stop portainer
   docker rm portainer
   ```
1. Repeat step 2 under `Installation`.

## Sources

- https://pimylifeup.com/raspberry-pi-portainer/
