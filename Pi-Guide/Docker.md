# Docker

Containerize packages for easy removal. Docker lets you remove an entire package cleanly — much harder with normally installed packages — and reduces the chance of packages conflicting with each other.

## Table of Contents

- [Installation](#installation)
- [Limit Container Log Size](#limit-container-log-size)
- [Testing](#testing)
- [Keeping Disk Usage in Check](#keeping-disk-usage-in-check)
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
1. Verify the user was added to the `docker` group:
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

## Limit Container Log Size

By default Docker keeps every line a container has ever logged, forever — a chatty container can slowly fill your SD card. Cap it for all containers:

1. Create Docker's config file:
   ```bash
   sudo nano /etc/docker/daemon.json
   ```
1. Paste the following in and save. This keeps at most 3 log files of 10 MB each per container:
   ```json
   {
     "log-driver": "json-file",
     "log-opts": {
       "max-size": "10m",
       "max-file": "3"
     }
   }
   ```
1. Restart Docker:
   ```bash
   sudo systemctl restart docker
   ```

<ins>IMPORTANT:</ins> this only applies to containers **created after** the change. Containers that already exist keep unlimited logs until you remove and recreate them (`docker compose up -d --force-recreate` for compose-based ones, or `docker rm` and re-run the `docker run` command). Check a container's actual setting with:

```bash
docker inspect --format '{{.HostConfig.LogConfig}}' [CONTAINERNAME]
```

An empty `map[]` means it's still on the old, unlimited setting.

## Testing

Test by running the Hello World container:

```bash
docker run hello-world
```

## Keeping Disk Usage in Check

Every time a container updates, the old image stays on disk unless something deletes it. Check now and then:

```bash
docker system df
```

The `RECLAIMABLE` column is space you can get back. If it's large, remove images no container is using:

```bash
docker image prune -a
```

If you run [Watchtower](/Pi-Guide/Watchtower.md) with `WATCHTOWER_CLEANUP=true`, old images are removed automatically after each update and this should stay near zero.

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
