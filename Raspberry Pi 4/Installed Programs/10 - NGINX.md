# NGINX
Create your own website hosted on your Pi.
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
### Editing the Website
1. To start editing the website, go to:
    ```
    sudo nano /var/www/html/index.html
    ```
    - Main Config: 
        ```
        sudo nano /etc/nginx/nginx.conf
        ```
    - Site Config: 
        ```
        sudo nano /etc/nginx/sites-available/default
        ```
2. After editing, make sure to run the below to verify the config works.
    ```
    sudo nginx -t
    ``` 
3. Finally, run the below to put the config into action.
    ```
    sudo /etc/init.d/nginx reload
    ```
### DDoS Protection
1. Edit the main config:
    ```
    sudo nano /etc/nginx/nginx.conf
    ```
2. Inside of http{}, enter the following at the bottom:
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
3. After editing, make sure to run the below to verify the config works.
    ```
    sudo nginx -t
    ``` 
4. Finally, run the below to put the config into action.
    ```
    sudo /etc/init.d/nginx reload
    ```
### Enable HTTPS
_DISCLAIMER 1_: If you have another app installed that uses port 80, try to see if you can change it's port to something else. If not, then HTTPS will not work as NGINX needs port 80.

_DISCLAIMER 2_: Set up the `Host Record` on your domain provider as so:
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
5. Create crontab to schedule renewable of certificates (do `sudo crontab -e` in addition to below):
    ```
    crontab -e
    ```
    ```
    0 12 * * * /usr/bin/certbot renew --quiet
    ```
### Blocking IP's
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
1. Visit `http://[PIIPADDRESS]`
## Troubleshooting
If you can't install NGINX, it's likely because you have something installed that is currently using port 80 on the Pi. Follow the below steps to change NGINX's port, however, doing so will render you unable to enable HTTPS on the website.
1. Edit the default configuration so NGINX listens to a different port (default is 80) to avoid conflicts:
    ```
    sudo nano /etc/nginx/sites-enabled/default
    ```
2. Edit the first two lines from 80 to 81 as shown below:
    ```
    listen 81 default_server;
    listen [::]:81 default_server;
    ```
## Sources
* https://pimylifeup.com/raspberry-pi-nginx/
* https://stackoverflow.com/questions/10829402/how-to-start-nginx-via-different-portother-than-80
* https://medium.com/l0rd/how-to-host-a-nginx-website-with-raspberry-pi-ddos-protected-1a166e36cce
* https://developpaper.com/compiling-sass-and-scss-with-nginx/
* https://varhowto.com/how-to-enable-https-for-nginx-websites-on-raspbian-raspberry-pi-certbot-python-certbot-nginx-automatic-raspbian/
* https://www.nginx.com/blog/using-free-ssltls-certificates-from-lets-encrypt-with-nginx/