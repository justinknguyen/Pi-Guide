# Docker

Containerize packages for easy removal. The benefit of Docker is being able to quickly and easily remove an entire package — something that's much harder to do with normally installed packages. There's also less chance of packages conflicting with each other.

## Table of Contents

- [Installation](#installation)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```bash
   sudo apt-get update && sudo apt-get upgrade
   ```
1. Install Docker:
   ```bash
   curl -sSL https://get.docker.com | sh
   ```
1. Add permissions for the current user:
   ```bash
   sudo usermod -aG docker ${USER}
   ```
1. Check it running with:
   ```bash
   groups ${USER}
   ```
1. Reboot:
   ```bash
   sudo reboot
   ```
1. Docker Compose (v2, the `docker compose` plugin) is already installed by the `get.docker.com` script above. Verify it:
   ```bash
   docker compose version
   ```
   - Avoid `sudo apt-get install docker-compose` — that installs the older, deprecated standalone `docker-compose` (v1) package with the hyphenated command syntax.
1. Enable the Docker system service to start your containers on boot:
   ```bash
   sudo systemctl enable docker
   ```

## Testing

Test by running the Hello World container:

```bash
docker run hello-world
```

## Troubleshooting

If you're unable to access sites that were installed with Docker, then Docker got corrupted somehow. Run the following commands, which remove and reinstall Docker:
```bash
sudo apt-get purge docker-ce docker-ce-cli containerd.io
sudo rm -fr /var/lib/containerd/
sudo apt-get install docker-ce docker-ce-cli containerd.io
```

## Sources

- https://dev.to/elalemanyo/how-to-install-docker-and-docker-compose-on-raspberry-pi-1mo
- https://github.com/docker/for-linux/issues/1178
