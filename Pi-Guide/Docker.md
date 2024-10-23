# Docker

Containerize packages for easy removal. The benefit of Docker is being able to quickly and easily remove the entire package, which is much harder than normally installed packages. There is also a less chance of packages conflicting with each other.

## Table of Contents

- [Installation](#installation)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```
   sudo apt-get update && sudo apt-get upgrade
   ```
2. Install Docker:
   ```
   curl -sSL https://get.docker.com | sh
   ```
3. Add a Non-Root User to the Docker Group:
   ```
   sudo usermod -aG docker pi
   ```
4. Then add permissions to the current user:
   ```
   sudo usermod -aG docker ${USER}
   ```
5. Check it running with:
   ```
   groups ${USER}
   ```
6. Reboot:
   ```
   sudo reboot
   ```
7. Install Docker-Compose:
   ```
   sudo apt-get install docker-compose
   ```
8. Enable the Docker system service to start your containers on boot:
   ```
   sudo systemctl enable docker
   ```

## Testing

Test by running the Hello World container:

```
docker run hello-world
```

## Troubleshooting

If you're unable to access sites that were installed with Docker, then Docker got corrupted somehow. Run the following commands which removes and reinstalls Docker:
```
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo rm -fr /var/lib/containerd/
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

## Sources

- https://dev.to/elalemanyo/how-to-install-docker-and-docker-compose-on-raspberry-pi-1mo
