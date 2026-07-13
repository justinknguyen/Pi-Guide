# Watchtower

Automatically updates your running Docker containers to the latest image — no more manual `docker pull` in each guide's Updating section.

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

## Testing

Trigger a one-off check and watch the logs:

```bash
docker exec watchtower /watchtower --run-once || docker logs watchtower
```

You should see it scan your containers and report "Session done" with counts of scanned/updated.

## Sources

- https://containrrr.dev/watchtower/
- https://immich.app/docs/install/docker-compose#step-4---upgrading
