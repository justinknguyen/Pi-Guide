# Watchtower

Automatically updates your running Docker containers to the latest image — no more manual `docker pull` in each guide's Updating section.

Note: the `containrrr/watchtower` project was **archived in December 2025** and its last release was in November 2023 — it won't get fixes, including for future Docker API changes. It still works today, but for a new setup consider updating containers manually with each guide's Updating section, or look for an actively maintained fork.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Excluding Containers](#excluding-containers)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

```bash
docker run -d --name watchtower --restart unless-stopped -e WATCHTOWER_CLEANUP=true -v /var/run/docker.sock:/var/run/docker.sock containrrr/watchtower
```

- Checks for new images every 24 hours by default. To change the schedule, add e.g. `-e WATCHTOWER_SCHEDULE="0 0 4 * * *"` (4 AM daily — note the 6-field cron format with seconds).
- `WATCHTOWER_CLEANUP=true` deletes old images after updating so they don't fill your SD card.

## Excluding Containers

Auto-updating isn't right for everything — some apps (notably [immich](/Pi-Guide/immich.md)) occasionally ship breaking releases that need manual migration steps. Exclude a container by adding a label to its compose file:

```yaml
services:
  immich-server:
    labels:
      - com.centurylinklabs.watchtower.enable=false
```

Or flip the model around — run Watchtower with `-e WATCHTOWER_LABEL_ENABLE=true` so it *only* updates containers you've labelled `com.centurylinklabs.watchtower.enable=true`.

Other containers worth thinking about before letting Watchtower update them:

- **[Jellyfin](/Pi-Guide/Jellyfin.md), or anything with plugins.** Plugins are built for a specific version, so an overnight update can leave one disabled until its author catches up. Auto-updating can still be the right call, as long as a missing plugin feature makes you check for an update first.
- **[Gluetun](/Pi-Guide/Gluetun.md), and any container other containers share a network with.** Updating gluetun recreates it, and the containers using its network (`network_mode: service:gluetun`) then fail to start until they're recreated too. Exclude it and update the whole group by hand.
- **Databases** (Postgres, MariaDB), which may need a manual step between major versions.

Updating also recreates the container, which is how existing containers pick up changes to Docker's defaults, like the [log size limit](/Pi-Guide/Docker.md#limit-container-log-size).

## Testing

Trigger a one-off check and watch the logs:

```bash
docker exec watchtower /watchtower --run-once || docker logs watchtower
```

You should see it scan your containers and report "Session done" with counts of scanned/updated.

## Sources

- https://containrrr.dev/watchtower/
- https://immich.app/docs/install/docker-compose#step-4---upgrading
