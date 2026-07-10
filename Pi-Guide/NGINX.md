# NGINX

Host your website on your Pi.

A much simpler alternative than setting nginx configs manually is [NGINX Proxy Manager](https://nginxproxymanager.com/).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Hosting the Website](#1-hosting-the-website)
  - [2. DDoS Protection](#2-ddos-protection)
  - [3. Enable HTTPS](#3-enable-https)
  - [4. Blocking IPs](#4-blocking-ips)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

If you have Pi-Hole installed, you will need to change its port, since NGINX will need to use the default port (80).

1. Go to:
   ```bash
   sudo nano /etc/lighttpd/lighttpd.conf
   ```
1. Change the line that says `server.port = 80` to `server.port = 8080`.
1. Go to:
   ```bash
   sudo nano /etc/lighttpd/external.conf
   ```
1. Enter the following in the file and save it:
   ```
   server.port := 8080
   ```
1. Restart the service:
   ```bash
   sudo service lighttpd restart
   ```
1. You can test and access Pi-Hole's webui using `[PIIPADDRESS]:8080`.

## Installation

1. Update and upgrade:
   ```bash
   sudo apt update
   sudo apt upgrade
   ```
1. Remove apache2, install NGINX, and start the service:
   ```bash
   sudo apt remove apache2
   sudo apt install nginx
   sudo systemctl start nginx
   ```

## Configuration

### 1. Hosting the Website

1. Assuming your website is in a GitHub repo, go to the directory, clone the repo, and check the folder name it was cloned to:
   ```bash
   cd /var/www/html/
   sudo git clone [your repo.git]
   ls
   ```
1. Edit the site config so it points to the git folder:
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
   - Find the line `root /var/www/html/` and append your git folder name to it, for example `root /var/www/html/my-website`.
1. Verify and reload nginx:
   ```bash
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

### 2. DDoS Protection

1. Edit the main config:
   ```bash
   sudo nano /etc/nginx/nginx.conf
   ```
1. Inside of `http{}`, enter the following at the bottom:
   ```nginx
           limit_req_zone $binary_remote_addr zone=global:10m rate=1r/m;
           limit_conn_zone $binary_remote_addr zone=addr:10m;
           server {
               location / {
                   limit_req zone=global burst=10 nodelay;
                   limit_conn addr 1;
                   limit_rate 100k;
               }
           }
   ```
1. Verify and reload nginx:
   ```bash
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

### 3. Enable HTTPS

First, forward ports 80 and 443 on your router. If you have another package installed that uses port 80, you will need to change its port number or see [Troubleshooting](#troubleshooting).

Set up the `Host Record` on your domain provider as so:

- Record 1 (allows users to reach your website with the address format: https://example.com):
  - Hostname: @
  - Address: your network's public IP address (https://whatismyipaddress.com/)
  - Record Type: if you chose your public IPv4 address, choose A. If you chose your IPv6, choose AAAA.
- Record 2 (allows users to reach your website with the address format: https://www.example.com):
  - Hostname: www
  - Address: your network's public IP address (https://whatismyipaddress.com/)
  - Record Type: if you chose your public IPv4 address, choose A. If you chose your IPv6, choose AAAA.

1. Ensure `server_name` is set:
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
   ```nginx
   server_name example.com www.example.com
   ```
   ```bash
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```
1. Install certbot:
   ```bash
   sudo apt-get install certbot python3-certbot-nginx
   ```
1. Do a dry run to make sure it works before proceeding:
   ```bash
   sudo certbot certonly --nginx --dry-run -d example.com -d www.example.com
   ```
1. If there are no errors, proceed:
   ```bash
   sudo certbot --nginx -d example.com -d www.example.com
   ```
1. Create a cron task to schedule renewal of certificates:
   ```bash
   sudo crontab -e
   ```
   ```
   0 12 * * * /usr/bin/certbot renew --quiet
   ```

### 4. Blocking IPs

If you notice a suspicious IP traffic, perform the following to block access from that IP.

1. Open the site config:
   ```bash
   sudo nano /etc/nginx/sites-available/default
   ```
1. Enter the following under `location`:
   ```nginx
           location / {
                   deny [IPADDRESS];
           }
   ```
1. Confirm config and reload:
   ```bash
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

## Testing

Visit `http://[PIIPADDRESS]`

## Troubleshooting

If you can't install NGINX, something else is likely already using port 80 on the Pi. Follow the steps below to change NGINX's port — note this may prevent you from enabling HTTPS.

If your `default` file has more than one server block, make sure each one has a `listen` line. Any block missing it will default to listening on port 80.

1. Edit the default configuration so NGINX listens to a different port (default is 80) to avoid conflicts:
   ```bash
   sudo nano /etc/nginx/sites-enabled/default
   ```
1. Edit the first two lines from 80 to 81 as shown below:
   ```nginx
   listen 81 default_server;
   listen [::]:81 default_server;
   ```
1. Confirm config and reload:
   ```bash
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

## Sources

- https://pimylifeup.com/raspberry-pi-nginx/
- https://stackoverflow.com/questions/10829402/how-to-start-nginx-via-different-portother-than-80
- https://medium.com/l0rd/how-to-host-a-nginx-website-with-raspberry-pi-ddos-protected-1a166e36cce
- https://varhowto.com/how-to-enable-https-for-nginx-websites-on-raspbian-raspberry-pi-certbot-python-certbot-nginx-automatic-raspbian/
- https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/
