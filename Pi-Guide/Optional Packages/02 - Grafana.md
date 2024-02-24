# Grafana

Monitor the Pi hardware. Most important information to me are CPU temp/load and storage.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Installation

1. Run everything as sudo:
   ```
   sudo su
   ```
2. Install Node Exporter:
   ```
   docker run -d  --net="host"  --pid="host"  -v "/:/host:ro,rslave"  quay.io/prometheus/node-exporter:latest  --path.rootfs=/host
   ```
3. You can test if Node Exporter is running by entering `[PIIPADDRESS]:9100` into your search bar.
4. Make a directory for Prometheus:
   ```
   mkdir Prometheus
   ```
5. cd into Prometheus:
   ```
   cd Prometheus/
   ```
6. Create a `prometheus.yml` file by entering:
   ```
   nano prometheus.yml
   ```
7. Copy and paste the following in then replace `PIIPADDRESS`:
   ```
   global:
   scrape_interval: 5s
   external_labels:
       monitor: 'node'
   scrape_configs:
   - job_name: 'prometheus'
       static_configs:
       - targets: ['PIIPADDRESS:9090'] ## IP Address of the localhost. Match the port to your container port
   - job_name: 'node-exporter'
       static_configs:
       - targets: ['PIIPADDRESS:9100'] ## IP Address of the localhost
   ```
8. To save the file, press `Ctrl+X` then `Y` then `Enter`.
9. Install Prometheus:
   ```
   docker run -d --name prometheus -p 9090:9090 -v /home/pi/Prometheus/prometheus.yml:/etc/prometheus/prometheus.yml prom/prometheus
   ```
10. Install Grafana:
    ```
    docker run -d --name=grafana -p 3000:3000 grafana/grafana
    ```

## Configuration

1. Login to Grafana by typing `[PIIPADDRESS]:3000` into your search bar. The default username and password is `admin`. Click on "Add your first data source" then select "Prometheus".
2. In the "URL" box, type in `http://[PIIPADDRESS]:9090` then click "Save & Test".
3. Google "Node Exporter Grafana Dashboards" and choose one (e.g., https://grafana.com/grafana/dashboards/11074).
4. Copy the ID (in this case, 11074).
5. Go back to Grafana, hover over the "+" icon on the left, and click "Import".
6. Paste in the ID and click "Load".
7. Under "VictoriaMetrics", select "Prometheus" and then import.
8. <ins>IMPORTANT:</ins> login to Portainer and click on your "Local" environment then "Containers". You should see node exporter, prometheus, and grafana installed. Click on each one and scroll down until you see "RESTART POLICIES" then set each to "Always". They will now start on their own whenever the Pi is rebooted.

## Testing

Gathering data metrics will take some time, so check back later and see if the data metrics are registering in your dashboard.

## Sources

- https://www.youtube.com/watch?v=83LWo7h_hvs&t=54s
