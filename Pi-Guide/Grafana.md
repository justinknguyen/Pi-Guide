# Grafana

Monitor the Pi hardware. The most important information to track is CPU temp/load and storage.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Prerequisites

[Docker](/Pi-Guide/Docker.md)

## Installation

1. Install Node Exporter:
   ```bash
   docker run -d --name=node-exporter --restart=unless-stopped --net="host" --pid="host" -v "/:/host:ro,rslave" quay.io/prometheus/node-exporter:latest --path.rootfs=/host
   ```
1. You can test if Node Exporter is running by entering `[PIIPADDRESS]:9100` into your address bar.
1. Make a directory for Prometheus and cd into it (this path must match the one mounted in the `docker run` command below):
   ```bash
   mkdir /home/pi/Prometheus
   cd /home/pi/Prometheus/
   ```
1. Create a `prometheus.yml` file by entering:
   ```bash
   nano prometheus.yml
   ```
1. Copy and paste the following in, then replace `<IP_ADDRESS>` with your Pi's IP address:
   ```yaml
   global:
     scrape_interval: 5s
     external_labels:
       monitor: 'node'
   scrape_configs:
     - job_name: 'prometheus'
       static_configs:
         - targets: ['<IP_ADDRESS>:9090'] # Replace <IP_ADDRESS> with your actual IP address
   
     - job_name: 'node-exporter'
       static_configs:
         - targets: ['<IP_ADDRESS>:9100'] # Replace <IP_ADDRESS> with your actual IP address
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Install Prometheus:
   ```bash
   docker run -d --name=prometheus --restart=unless-stopped -p 9090:9090 -v /home/pi/Prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
   ```
1. Install Grafana:
    ```bash
    docker run -d --name=grafana --restart=unless-stopped -p 3000:3000 grafana/grafana
    ```
   - The `--restart=unless-stopped` flag on each container makes them start on their own whenever the Pi reboots.
1. If you use the [UFW firewall](/Pi-Guide/SSH-Hardening.md#firewall-ufw), allow Prometheus and Grafana to reach the Pi's IP. They connect to `[PIIPADDRESS]:9090` and `:9100` from inside Docker, and UFW treats that as incoming traffic from Docker's network:
   ```bash
   sudo ufw allow from 172.16.0.0/12 to any port 9090 proto tcp comment 'Prometheus (Docker)'
   sudo ufw allow from 172.16.0.0/12 to any port 9100 proto tcp comment 'node-exporter (Docker)'
   ```
   If `docker network inspect` shows the container on a network outside `172.16.0.0/12`, add the same rule for that subnet — see [the note in SSH Hardening](/Pi-Guide/SSH-Hardening.md#firewall-ufw). Without these rules, Prometheus's Status → Targets page shows the targets as down, and Grafana's "Save & Test" times out.

## Configuration

1. Log in to Grafana by typing `[PIIPADDRESS]:3000` into your address bar. The default username and password are `admin`. Click on "Add your first data source" then select "Prometheus".
1. In the "URL" box, type in `http://[PIIPADDRESS]:9090` then click "Save & Test".
1. Google "Node Exporter Grafana Dashboards" and choose one (e.g., https://grafana.com/grafana/dashboards/11074 or https://grafana.com/grafana/dashboards/1860-node-exporter-full/).
1. Copy the ID (in this case, 11074).
1. Go back to Grafana, click on the "+" icon on the top-right, and click "Import dashboard".
1. Paste in the ID and click "Load".
1. Under "VictoriaMetrics", select "Prometheus" and then import.

## Testing

Gathering metrics takes some time, so check back later to see if data is registering in your dashboard.

## Sources

- https://www.youtube.com/watch?v=83LWo7h_hvs&t=54s
