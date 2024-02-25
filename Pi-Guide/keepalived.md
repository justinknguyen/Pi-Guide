# keepalived

keepalived is to have a High Availability setup between two Pi's, meaning, one Pi will act as the master server and the other will act as the slave server. When the master server shuts down, internet traffic will be redirected to the slave server until the master server comes back online.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- [Pi-Hole](/Pi-Guide/Pi-Hole.md)
- [Unbound](/Pi-Guide/Unbound.md) (recommended)
- [Gravity Sync](/Pi-Guide/Gravity%20Sync.md) (recommended)

## Installation

1.  Update:
    ```
    sudo apt update
    ```
2.  Install keepalived on both Pi's:
    ```
    sudo apt install keepalived
    ```
3.  Install libipset13 on both Pi's:
    ```
    sudo apt install libipset13
    ```
4.  Enable keepalived on both Pi's:
    ```
    sudo systemctl enable keepalived.service
    ```
5.  We will now create the pihole-FTL service check script. Run the following commands on both Pi's:
    ```
    sudo mkdir /etc/scripts
    sudo nano /etc/scripts/chk_ftl
    ```
6.  Copy and paste the following in:

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

7.  To save the file, press `Ctrl+X` then `Y` then `Enter`.
8.  Enter on both Pi's:
    ```
    sudo chmod 755 /etc/scripts/chk_ftl
    ```
9.  Now we will add the keepalived configuration on the primary Pi-hole machine, the Master/Active server. Change the settings according to your setup.
    ```
    sudo nano /etc/keepalived/keepalived.conf
    ```
10. Copy and paste the following in (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):

    ````
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
    ````

- router_id: should be an unique name, for instance your Pi-hole hostname
- state: describes which server is the Master/Active and which is the Backup/Standby server.
- interface: change this according to your network interface (e.g. eth0, ens3 etc)
- virtual_router_id: this can be any number between 0 and 255. Must be the same on the Master and Backup configs.
- priority: the master server should have a higher priority than the backup server.
- unicast_src_ip: should be the IP address of the (source) server.
- unicast_peer: should be the IP address of the other (peer) server.
- auth_pass: create your own (max 8 character) password. Must be the same on the Master and Backup configs.
- virtual_ipaddress: this will be the High Availability IP address.

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

## Configuration

Head into your router settings and replace the DNS servers for Pi-Hole with the virtual IP address you configured with keepalived.

## Testing

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
4. In our case, there is a noticeable time difference between the ping times. This is because one Pi is connected by ethernet, whereas the other is not. This makes it easy to see the switching between the two Pi's in real time. If you can't tell the difference between the two Pi's by this method, you can manually check the logs of each by entering `sudo systemctl status keepalived.service`. Start by pinging our virtual IP address constantly by entering:
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

## Troubleshooting

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
   `   sudo service pihole-FTL restart 
sudo systemctl restart dhcpcd 
sudo service unbound restart
sudo systemctl restart keepalived.service`
   **Important Note** <br>
   I sometimes experience a problem where the second Pi gets queries even if the keepalived service is stopped. Try removing the IPv6 vrrp block from the keepalived config file, then test for any more queries. If you see no more queries, then add it back and see if it happens again. <br><br>
   If it happens again, then it could be an IPv6 issue. The reason is because of a file in your router when setting up [Unbound](/Pi-Guide/Unbound.md). The fix would be to keep the file, but instead of the IP addresses of the Pi's, enter the IPv6 vrrp address when we setup keepalived. You should also disable router advertisement for IPv6 in router settings, if you have it on, then re-enable it after applying the IPv6 vrrp in the router file and see if everything is ok.

### Sources

- https://www.youtube.com/watch?v=IFVYe3riDRA
- https://www.youtube.com/watch?v=hPfk0qd4xEY&t=675s
- https://www.reddit.com/r/pihole/comments/d5056q/tutorial_v2_how_to_run_2_pihole_servers_in_ha/?sort=new
