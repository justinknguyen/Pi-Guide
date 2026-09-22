# Docker

Containerize packages for easy removal. Docker lets you remove an entire package cleanly — much harder with normally installed packages — and reduces the chance of packages conflicting with each other.

## Table of Contents

- [Installation](#installation)
- [Limit Container Log Size](#limit-container-log-size)
- [Testing](#testing)
- [Compose Project Habits](#compose-project-habits)
  - [Keep Secrets in a `.env` File](#keep-secrets-in-a-env-file)
  - [Give Each Project a Fixed Name](#give-each-project-a-fixed-name)
  - [Stop, Don't Down](#stop-dont-down)
- [Permissions on Container Data](#permissions-on-container-data)
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

## Compose Project Habits

### Keep Secrets in a `.env` File

Passwords, API keys and VPN keys don't belong in `docker-compose.yml`. Compose files get copied around: into backups, onto a NAS, pasted into a forum post when something breaks. Compose automatically reads a file called `.env` in the same folder, so put the values there:

```bash
# ~/myapp/.env
DB_PASSWORD=long-random-value
```

```yaml
# ~/myapp/docker-compose.yml
    environment:
      - DB_PASSWORD=${DB_PASSWORD}
```

Then make both files readable only by you:

```bash
chmod 600 ~/myapp/.env ~/myapp/docker-compose.yml
```

Things to know:

- **A missing `.env` doesn't stop anything.** Compose prints a warning (`The "DB_PASSWORD" variable is not set. Defaulting to a blank string.`) and starts the container with an empty value. If an app suddenly loses its login, or can't decrypt its own saved settings, check the `.env` is still there and readable. You can check a value reached the container without printing it: `docker exec [CONTAINERNAME] sh -c 'echo ${#DB_PASSWORD}'` shows its length.
- **`docker compose config` prints the file with every secret filled in.** Don't paste its output anywhere.
- A backup still carries the `.env` file (at mode 600). That's usually what you want, since you'll need it to restore. It just means the backup deserves the same care as the Pi.
- **Moving a secret into `.env` isn't the same as changing it.** If a key was ever in a file that got shared or backed up somewhere you don't control, generate a new one.

### Give Each Project a Fixed Name

Compose names a project after its folder, and prefixes the project's volumes and networks with that name. `~/myapp` gets a volume called `myapp_data`. Rename or move the folder and the next `docker compose up` creates a **new, empty** `newname_data` volume. The app then starts as if freshly installed, while your data sits untouched in the old volume.

Pin the name at the top of every compose file:

```yaml
name: myapp
services:
  ...
```

With that, you can reorganise folders freely: the project, its volumes and its networks keep their names. (Folders mounted by path, like `/home/pi/myapp/config:/config`, aren't affected either way. Only named volumes are.)

### Stop, Don't Down

`docker compose down` removes the containers **and the project's network**. The next `up` creates a new network, possibly on a different address range. That matters once you have [firewall rules](/Pi-Guide/SSH-Hardening.md#firewall-ufw) that allow Docker's networks by range: the rule no longer matches, and the connections it allowed silently stop working.

- To stop a project: `docker compose stop` (and `docker compose start`).
- To apply a changed compose file: `docker compose up -d`, which recreates only what changed and keeps the network.
- Save `down` for when you're removing the project.

## Permissions on Container Data

Before running a recursive `chmod` on folders that containers use (tightening permissions is a common cleanup), check **which user the container runs as** and **who owns the folder**:

```bash
docker inspect [CONTAINERNAME] --format '{{.Config.User}}'   # empty = root
docker exec [CONTAINERNAME] id                              # what it actually runs as
stat -c '%U:%G %a' ~/myapp/config
```

The trap: a folder owned by `root:root` with mode `755`, used by a container running as uid `1000`. The container can only get in through the last digit, the "everyone else" permissions. Remove those (`chmod -R o-rwx`) and the app is locked out of its own config. On one server this took down Jellyfin and another container, both the same way, with only a `503` or "permission denied" to go on.

- **Fix it by ownership, not by opening permissions:** `sudo chown -R 1000:1000 ~/myapp/config` (or whatever uid the container uses). Then the owner permissions apply and you can safely remove everyone else's.
- LinuxServer.io images (`lscr.io/linuxserver/...`) run as the `PUID`/`PGID` from their environment, even though `.Config.User` looks empty. `docker exec ... id` shows the truth.
- Before a sweeping change, save the current permissions so you can put them back:
  ```bash
  sudo find ~ -xdev -printf '%m %u:%g %p\n' > ~/perms-before.txt
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
