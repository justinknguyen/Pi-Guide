# NGINX

Host your website on your Pi.

A much simpler alternative than setting nginx configs manually is [NGINX Proxy Manager](https://nginxproxymanager.com/)

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Hosting the Website](#1-hosting-the-website)
  - [2. DDoS Protection](#2-ddos-protection)
  - [3. Enable HTTPS](#3-enable-https)
  - [4. Blocking IP's](#4-blocking-ips)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

If you have Pi-Hole installed, you will need to change the port as this will now need to use the default port (80).

1. Go to:
   ```
   sudo nano /etc/lighttpd/lighttpd.conf
   ```
2. Change the line that says `server.port = 80` to `server.port = 8080`.
3. Go to:
   ```
   sudo nano /etc/lighttpd/external.conf
   ```
4. Enter the following in the file and save it:
   ```
   server.port := 8080
   ```
5. Restart the service:
   ```
   sudo service lighttpd restart
   ```
6. You can test and access Pi-Hole's webui using `[PIIPADDRESS]:8080`.

## Installation

1. Update and upgrade:
   ```
   sudo apt update
   sudo apt upgrade
   ```
2. Remove apache2:
   ```
   sudo apt remove apache2
   ```
3. Install NGINX:
   ```
   sudo apt install nginx
   ```
4. Start the service:
   ```
   sudo systemctl start nginx
   ```

## Configuration

### 1. Hosting the Website

1. Assuming you created the Website and have it on your GitHub repo, go to the directory:
   ```
   cd /var/www/html/
   ```
2. Then clone your repo into the path:
   ```
   sudo git clone [your repo.git]
   ```
3. Check the name of the folder where your repo was cloned to:
   ```
   ls
   ```
4. Edit the site config so it points to the git folder:
   ```
   sudo nano /etc/nginx/sites-available/default
   ```
   - Find the line `root /var/www/html/` and append your git folder name to it, for example `root /var/www/html/my-website`.
5. Verify and reload nginx:
   ```
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

### 2. DDoS Protection

1. Edit the main config:
   ```
   sudo nano /etc/nginx/nginx.conf
   ```
2. Inside of `http{}`, enter the following at the bottom:
   ```
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
3. Verify and reload nginx:
   ```
   sudo nginx -t
   sudo /etc/init.d/nginx reload
   ```

### 3. Enable HTTPS

First, forward port 80 and 443 on your router. If you have another package installed that uses port 80, you will need to change it's port number or see [Troubleshooting](#troubleshooting).

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
   ```
   sudo nano /etc/nginx/sites-available/default
   ```
   ```
   server_name example.com www.example.com
   ```
   ```
   sudo nginx -t
   ```
   ```
   sudo /etc/init.d/nginx reload
   ```
2. Install certbot:
   ```
   sudo apt-get install certbot python3-certbot-nginx
   ```
3. Do a dry run to make sure it works before proceeding:
   ```
   sudo certbot certonly --nginx --dry-run -d example.com -d www.example.com
   ```
4. If there are no errors, proceed:
   ```
   sudo certbot --nginx -d example.com -d www.example.com
   ```
5. Create a cron task to schedule renewal of certificates:
   ```
   sudo crontab -e
   ```
   ```
   0 12 * * * /usr/bin/certbot renew --quiet
   ```

### 4. Blocking IP's

If you notice a suspicious IP traffic, perform the following to block access from that IP.

1. Open the site config:
   ```
   sudo nano /etc/nginx/sites-available/default
   ```
2. Enter the following under `location`:
   ```
           location / {
                   deny [IP ADDRESS];
           }
   ```
3. Confirm config and reload:
   ```
   sudo nginx -t
   ```
   ```
   sudo /etc/init.d/nginx reload
   ```

## Testing

Visit `http://[PIIPADDRESS]`

## Troubleshooting

If you can't install NGINX, it's likely because you have something installed that is currently using port 80 on the Pi. Follow the below steps to change NGINX's port, however, doing so might render you unable to enable HTTPS on the website.

In your `default` file, when editing the ports, if you have more than one server block, you need to make sure your have the `listen` command in each server block. If you don't have it in one of them, it will default to listening on port 80.

1. Edit the default configuration so NGINX listens to a different port (default is 80) to avoid conflicts:
   ```
   sudo nano /etc/nginx/sites-enabled/default
   ```
2. Edit the first two lines from 80 to 81 as shown below:
   ```
   listen 81 default_server;
   listen [::]:81 default_server;
   ```
3. Confirm config and reload:
   ```
   sudo nginx -t
   ```
   ```
   sudo /etc/init.d/nginx reload
   ```

## Sources

- https://pimylifeup.com/raspberry-pi-nginx/
- https://stackoverflow.com/questions/10829402/how-to-start-nginx-via-different-portother-than-80
- https://medium.com/l0rd/how-to-host-a-nginx-website-with-raspberry-pi-ddos-protected-1a166e36cce
- https://developpaper.com/compiling-sass-and-scss-with-nginx/
- https://varhowto.com/how-to-enable-https-for-nginx-websites-on-raspbian-raspberry-pi-certbot-python-certbot-nginx-automatic-raspbian/
- https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/
