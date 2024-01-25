# Pi-Hole, Gravity Sync, and keepalived
Network-wide ad-blocking with recursive DNS and full IPv4/IPv6 support. The Pi 4 acts as the master server, while the Pi Zero 2 W acts as the slave server (i.e., when the master is down, internet traffic will be redirected to the slave). <br><br>
**Disclaimer**<br>
To install and setup Pi-Hole, follow the guide outlined under the `Raspberry Pi 4` folder for Pi-Hole. If you have Unbound on your primary Pi, install and configure Unbound on your second Pi also. One thing you should keep in mind when installing Unbound on the second Pi is the port. In the Unbound config file, change the port to `5353`. <br>

The following guide will show how to setup a second Pi as a slave server and sync between the two Pi's.
## Installation (Gravity Sync)
This install guide will begin by setting up syncing between the two Pi's. Adding adlists or whitelists in one Pi will automatically add them to the other Pi.
1. Match your settings in Pi-Hole in your second Pi with your primary Pi.
2. Disable DHCP in Pi-Hole settings.
3. SSH into second Pi and enter:
    ```
    sudo apt update && sudo apt install sqlite3 sudo git rsync ssh
    ```
4. Ensure passwordless sudo. On secondary Pi, enter:
    ```
    sudo EDITOR=nano visudo
    ```
5. Scroll down to where you see `%sudo ALL=(ALL:ALL) ALL` and comment it out with `#`. Enter the following underneath it:
    ```
    %sudo ALL=(ALL:ALL) NOPASSWD:ALL
    ```
6. To save the file, press `Ctrl+X` then `Y` then `Enter`.
7. Reboot:
    ```
    sudo reboot
    ```
8. Repeat Steps 3-7 on your primary Pi.
9. On both Pi's, enter:
    ```
    curl -sSL https://raw.githubusercontent.com/vmstan/gs-install/main/gs-install.sh | bash
    ```
10. You will receive prompts to enter an IP Address and host username. This is NOT the host name of the Pi, but the username you use to log in. On the primary Pi, enter the secondary Pi's IP Address and host username (pi), if it says the authenticity of the host can't be established, enter `yes` in the following prompt. On the secondary Pi, enter the primary Pi's IP Address and host username (pi).
## Configuration (Gravity Sync)
1. Check if there are any differences between the two Pi's in Pi-Hole by entering in the second Pi:
    ```
    gravity-sync compare
    ```
2. Pull settings from primary Pi to secondary Pi by entering in the second Pi:
    ```
    gravity-sync pull
    ```
3. You should now see adlists and whitelists synchronised between the two Pi's.
4. Next is to automate this syncing between the two Pi's on a schedule:
    ```
    gravity-sync auto
    ```
5. Optional: set sync frequency to 15 mins.
    ```
    gravity-sync auto quad
    ```
## Troubleshooting (Gravity Sync)
- If you messed up the configuration, enter the following to restart the config:
    ```
    gravity-sync config
    ```
- If you need to completely restart, remove Gravity Sync by entering the following:
    ```
    gravity-sync purge
    ```
- If pushing and pulling is failing, check if the rsync versions are the same between both Pi's:
    ```
    rsync --version
    ```
    1. If not, it's likely one is on v3.2.3, and in order to update to the latest (v3.2.7 currently), install dependencies:
       ```
       sudo apt install gcc g++ gawk autoconf automake python3-cmarkgfm libssl-dev attr libxxhash-dev libattr1-dev liblz4-dev libzstd-dev acl libacl1-dev -y
       ```
    1. Download the latest rsync file:
       ```
       wget https://download.samba.org/pub/rsync/src/rsync-3.2.7.tar.gz
       ```
    1. Extract the file:
       ```
       tar -xf rsync-3.2.7.tar.gz
       ```
    1. Configure rsync:
       ```
       cd rsync-3.2.7
       ./configure
       ```
    1. Prepare rsync install files:
       ```
       make
       ```
    1. Install rsync:
       ```
       sudo make install
       ```
    1. Reboot and confirm the rsync version:
       ```
       sudo reboot
       ```
## Installation (keepalived)
keepalived is to have a High Availability setup between two Pi's, meaning, one Pi will act as the master server and the other will act as the slave server. When the master server shuts down, internet traffic will be redirected to the slave server until the master server comes back online.
1. Update:
    ```
    sudo apt update
    ```
2. Install keepalived on both Pi's:
    ```
    sudo apt install keepalived
    ```
3. Install libipset13 on both Pi's:
    ```
    sudo apt install libipset13
    ```
4. Enable keepalived on both Pi's:
    ```
    sudo systemctl enable keepalived.service
    ```
5. We will now create the pihole-FTL service check script. Run the following commands on both Pi's:
    ```
    sudo mkdir /etc/scripts
    sudo nano /etc/scripts/chk_ftl
    ```
6. Copy and paste the following in:
    ```
    #!/bin/sh

    STATUS=$(ps ax | grep -v grep | grep pihole-FTL)

    if [ "$STATUS" != "" ]
    then
        exit 0
    else
        exit 1
    fi
    ```
7. To save the file, press `Ctrl+X` then `Y` then `Enter`.
8. Enter on both Pi's:
    ```
    sudo chmod 755 /etc/scripts/chk_ftl
    ```
9. Now we will add the keepalived configuration on the primary Pi-hole machine, the Master/Active server. Change the settings according to your setup.
    ```
    sudo nano /etc/keepalived/keepalived.conf
    ```
10. Copy and paste the following in (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):
    ```
    global_defs {
        router_id pihole-dns-01
        script_user root
        enable_script_security
    }
    
    vrrp_script chk_ftl {
        script "/etc/scripts/chk_ftl"
        interval 1
        weight -10
    }
    
    vrrp_instance PIHOLE {
        state MASTER
        interface eth0
        virtual_router_id 55
        priority 150
        advert_int 1
        unicast_src_ip [PRIMARYPIIPADDRESS]
        unicast_peer {
            [SECONDARYPIIPADDRESS]
        }
    
        authentication {
            auth_type PASS
            auth_pass xxXXxxXX
        }
    
        virtual_ipaddress {
            [LANIPv4PREFIX].20/24
        }
    
        track_script {
            chk_ftl
        }
    }
    ```
Explanation of the options:
* router_id: should be an unique name, for instance your Pi-hole hostname
* state: describes which server is the Master/Active and which is the Backup/Standby server.
* interface: change this according to your network interface (e.g. eth0, ens3 etc)
* virtual_router_id: this can be any number between 0 and 255. Must be the same on the Master and Backup configs.
* priority: the master server should have a higher priority than the backup server.
* unicast_src_ip: should be the IP address of the (source) server.
* unicast_peer: should be the IP address of the other (peer) server.
* auth_pass: create your own (max 8 character) password. Must be the same on the Master and Backup configs.
* virtual_ipaddress: this will be the High Availability IP address.
11. If you have IPv6 enabled on Pi-Hole, enter the following after everything above. I suggest you double-check your IPv6 address' with `ifconfig`. (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):
    ```
    vrrp_instance PIHOLEv6 {
        state MASTER
        interface eth0
        virtual_router_id 65
        priority 150
        advert_int 1
        unicast_src_ip [PRIMARYPIIPv6ADDRESS]
        unicast_peer {
            [SECONDARYPIIPv6ADDRESS]
        }
    
        authentication {
            auth_type PASS
            auth_pass xxXXxxXX
        }
    
        virtual_ipaddress {
            [LANIPv6PREFIX]::20/64
        }
    
        track_script {
            chk_ftl
        }
    }
    ```
12. Now we will add the keepalived configuration on the secondary Pi-hole machine, the Backup/Standby server.
    ```
    sudo nano /etc/keepalived/keepalived.conf
    ```
13. Copy and paste the following in (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):
    ```
    global_defs {
        router_id pihole-dns-02
        script_user root
        enable_script_security
    }
    
    vrrp_script chk_ftl {
        script "/etc/scripts/chk_ftl"
        interval 1
        weight -10
    }
    
    vrrp_instance PIHOLE {
        state BACKUP
        interface wlan0
        virtual_router_id 55
        priority 100
        advert_int 1
        unicast_src_ip [SECONDARYPIIPADDRESS]
        unicast_peer {
            [PRIMARYPIIPADDRESS]
        }
        
        authentication {
            auth_type PASS
            auth_pass xxXXxxXX
        }
        
        virtual_ipaddress {
            [LANIPv4PREFIX].20/24
        }
        
        track_script {
            chk_ftl
        }
    }
    ```
14. If you have IPv6 enabled on Pi-Hole, enter the following after everything above. I suggest you double-check your IPv6 address' with `ifconfig`. (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):
    ```
    vrrp_instance PIHOLEv6 {
        state BACKUP
        interface wlan0
        virtual_router_id 65
        priority 100
        advert_int 1
        unicast_src_ip [SECONDARYPIIPv6ADDRESS]
        unicast_peer {
            [PRIMARYPIIPv6ADDRESS]
        }
        
        authentication {
            auth_type PASS
            auth_pass xxXXxxXX
        }
        
        virtual_ipaddress {
            [LANIPv6PREFIX]::20/64
        }
        
        track_script {
            chk_ftl
        }
    }
    ```
15. Restart keepalived on both Pi's:
    ```
    sudo systemctl restart keepalived.service
    ```
## Configuration (keepalived)
Head into your router settings and replace the DNS servers for Pi-Hole with the virtual IP address you configured with keepalived.
## Testing (keepalived)
1. Check the status of keepalived on both Pi's:
    ```
    sudo systemctl status keepalived.service
    ```
2. In your command terminal on Windows, check if both Pi's are responding by pinging them:
    ```
    ping [PRIMARYPIIPADDRESS]
    ping [SECONDARYPIIPADDRESS]
    ```
3. Now, ping the virtual IP address:
    ```
    ping [VIRTUALIPADDRESS]
    ```
4. In our case, there is a noticeable time difference between the ping from Pi 4 and Pi Zero 2 W. This is because the Pi 4 is connected by ethernet, whereas the Pi Zero 2 W is not. This makes it easy to see the switching between the two Pi's in real time. If you can't tell the difference between the two Pi's by this method, you can manually check the logs of each by entering `sudo systemctl status keepalived.service`. Start by pinging our virtual IP address constantly by entering:
    ```
    ping [VIRTUALIPADDRESS] -t
    ```
5. Now, `sudo reboot` the primary Pi and you can see the time jump up randomly; the virtual IP address switched to our secondary Pi. Once the primary Pi boots back up, we can see the time lower and become consistent. If your virtual IP address does not switch back to the primary Pi, follow the Troubleshooting steps below. 
6. Instead of rebooting to test, you could also stop and start the service on either Pi's:
    ```
    sudo systemctl stop keepalived.service
    ```
    ```
    sudo systemctl start keepalived.service
    ```
## Troubleshooting (keepalived)
In my case, the virtual IP address does not switch back to the primary Pi after a reboot without restarting keepalived with `sudo systemctl restart keepalived.service`. So, all we have to do is make a crontask to automate this restart when the Pi boots back up.
1. Enter on both Pi's (try without `sudo` also as files are different between the two for somereason):
    ```
    sudo crontab -e
    ```
2. Copy and paste the following in at the bottom:
    ```
    @reboot sleep 60 && sudo systemctl restart keepalived.service
    ```
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
<!-- -->
<br>

Unsure if this is needed, but I've also set a delay for when Pi-Hole starts up after a reboot by going into:
```
sudo nano /etc/pihole/pihole-FTL.conf
```
1. Then copy and paste the following in:
    ```
    DELAY_STARTUP=5
    ```
2. Another delay was added inside the keepalived config file:
    ```
    sudo nano /etc/keepalived/keepalived.conf
    ```
3. Then copy and paste the following in under `enable_script_security` at the top:
    ```
    vrrp_startup_delay 5.5
    ```
4. Finally, restart services:
    ```
    sudo service pihole-FTL restart 
    sudo systemctl restart dhcpcd 
    sudo service unbound restart
    sudo systemctl restart keepalived.service
    ```
**Important Note** <br>
I sometimes experience a problem where the second Pi gets queries even if the keepalived service is stopped. Try removing the IPv6 vrrp block from the keepalived config file, then test for any more queries. If you see no more queries, then add it back and see if it happens again. <br><br> 
If it happens again, then it’s definitely an IPv6 problem and the reason is because of the file in our router when setting up Unbound [here](https://github.com/justinknguyen/PiGuide/wiki/Raspberry-Pi-4#getting-ipv6-to-work-with-unbound). The fix would be to keep the file but instead of the IP addresses of the Pi's, enter the IPv6 vrrp address when we setup keepalived. You should also disable router advertisement for IPv6 in router settings (this will render IPv6 unusable) then renable it after applying the IPv6 vrrp in the router file and seeing if everything seems ok.

### Sources
* https://www.youtube.com/watch?v=IFVYe3riDRA
* https://github.com/vmstan/gravity-sync
* https://www.youtube.com/watch?v=hPfk0qd4xEY&t=675s
* https://www.reddit.com/r/pihole/comments/d5056q/tutorial_v2_how_to_run_2_pihole_servers_in_ha/?sort=new
* https://linuxhint.com/update-rsync-raspberry-pi/
