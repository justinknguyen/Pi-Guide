# keepalived

keepalived is used to create a High Availability setup between two Pis, meaning one Pi will act as the master server and the other will act as the backup server. When the master server shuts down, internet traffic will be redirected to the backup server until the master server comes back online.

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
    ```bash
    sudo apt update
    ```
1.  Install keepalived on both Pis:
    ```bash
    sudo apt install keepalived
    ```
1.  Install the ipset runtime library on both Pis (package name depends on OS version — `libipset13` on Raspberry Pi OS "Bookworm"/Debian 12, `libipset13t64` on "Trixie"/Debian 13+):
    ```bash
    sudo apt install libipset13 || sudo apt install libipset13t64
    ```
1.  Enable keepalived on both Pis:
    ```bash
    sudo systemctl enable keepalived.service
    ```
1.  We will now create the pihole-FTL service check script. Run the following commands on both Pis:
    ```bash
    sudo mkdir /etc/scripts
    sudo nano /etc/scripts/chk_ftl
    ```
1.  Copy and paste the following in (thanks to @Racerx323 for the updated script):

    ```bash
    #!/bin/bash
    STATUS=$(systemctl show -p ActiveState pihole-FTL | cut -d'=' -f2)
    STATUS1=$(systemctl show -p ActiveState unbound | cut -d'=' -f2)
    
    if [ "$STATUS" = "active" ] && [ "$STATUS1" = "active" ]
    then
        exit 0
    else
        exit 1
    fi
    ```

1.  To save the file, press `Ctrl+X` then `Y` then `Enter`.
1.  Enter on both Pis:
    ```bash
    sudo chmod 755 /etc/scripts/chk_ftl
    ```
1.  Now we will add the keepalived configuration on the primary Pi-Hole machine, the master/active server. Change the settings according to your setup.
    ```bash
    sudo nano /etc/keepalived/keepalived.conf
    ```
1. Copy and paste the following in (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):

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
- router_id: should be a unique name, for instance your Pi-hole hostname
- state: describes which server is the Master/Active and which is the Backup/Standby server.
- interface: change this according to your network interface (e.g. eth0, ens3 etc)
- virtual_router_id: this can be any number between 0 and 255. Must be the same on the Master and Backup configs.
- priority: the master server should have a higher priority than the backup server.
- unicast_src_ip: should be the IP address of the (source) server.
- unicast_peer: should be the IP address of the other (peer) server.
- auth_pass: create your own (max 8 character) password. Must be the same on the Master and Backup configs.
- virtual_ipaddress: this will be the High Availability IP address.

1. If you have IPv6 enabled on Pi-Hole, enter the following after everything above. I suggest you double-check your IPv6 addresses with `ifconfig`. (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):

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

1. Now we will add the keepalived configuration on the secondary Pi-hole machine, the Backup/Standby server.
    ```bash
    sudo nano /etc/keepalived/keepalived.conf
    ```
1. Copy and paste the following in (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):

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

1. If you have IPv6 enabled on Pi-Hole, enter the following after everything above. I suggest you double-check your IPv6 addresses with `ifconfig`. (<ins>IMPORTANT:</ins> the password you enter below has a maximum of 8 characters):

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

1. Restart keepalived on both Pis:
    ```bash
    sudo systemctl restart keepalived.service
    ```

## Configuration

1. Head into your router settings and replace the DNS servers for Pi-Hole with the virtual IP address you configured with keepalived.
1. Set a crontask to restart the keepalived service after a reboot. Enter on both Pis:
   ```bash
   sudo crontab -e
   ```
1. Copy and paste the following in at the bottom:
   ```bash
   @reboot sleep 60 && sudo systemctl restart keepalived.service
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. On both Pis, set a delay for when Pi-Hole starts up after a reboot. Pi-hole v6 replaced `/etc/pihole/pihole-FTL.conf` with `/etc/pihole/pihole.toml`, so set this via the CLI instead of editing a conf file directly:
   ```bash
   sudo pihole-FTL --config misc.delay_startup 5
   ```
   (or edit `/etc/pihole/pihole.toml` directly and add `delay_startup = 5` under the `[misc]` section)
1. On both Pis, add another delay inside the keepalived config file:
   ```bash
   sudo nano /etc/keepalived/keepalived.conf
   ```
1. Then copy and paste the following in under `enable_script_security` at the top:
   ```
   vrrp_startup_delay 5.5
   ```
1. Finally, restart services:
   ```bash
   sudo service pihole-FTL restart 
   sudo systemctl restart dhcpcd 
   sudo service unbound restart
   sudo systemctl restart keepalived.service
   ```

## Testing

1. Check the status of keepalived on both Pis:
   ```bash
   sudo systemctl status keepalived.service
   ```
1. In your command terminal on Windows, check if both Pis are responding by pinging them:
   ```bash
   ping [PRIMARYPIIPADDRESS]
   ping [SECONDARYPIIPADDRESS]
   ```
1. Now, ping the virtual IP address:
   ```bash
   ping [VIRTUALIPADDRESS]
   ```
1. In our case, there is a noticeable time difference between the ping times. This is because one Pi is connected by ethernet, whereas the other is not. This makes it easy to see the switching between the two Pis in real time. If you can't tell the difference between the two Pis by this method, you can manually check the logs of each by entering `sudo systemctl status keepalived.service`. Start by pinging our virtual IP address constantly by entering:
   ```bash
   ping [VIRTUALIPADDRESS] -t
   ```
1. Now, `sudo reboot` the primary Pi and you can see the time jump up randomly; the virtual IP address switched to our secondary Pi. Once the primary Pi boots back up, we can see the time lower and become consistent. If your virtual IP address does not switch back to the primary Pi, follow the Troubleshooting steps below.
1. Instead of rebooting to test, you could also stop and start the service on either Pi:
   ```bash
   sudo systemctl stop keepalived.service
   ```
   ```bash
   sudo systemctl start keepalived.service
   ```

## Troubleshooting

If you have a problem where the second Pi gets queries even if the keepalived service is stopped:
- Try removing the IPv6 vrrp block from the keepalived config file in both Pis, then test if you get more queries. If you see no more queries, then add it back and see if it happens again.
- If it happens again, then it could be an IPv6 issue
    - double check both of your Pis' IPv6 addresses with `ifconfig`. If it changed, update it inside of the keepalived config file.
    - try disabling and re-enabling router advertisement for IPv6 in router settings.
    - one reason is because of a file in your router when setting up [Unbound](/Pi-Guide/Unbound.md#this-may-help). The fix would be to keep the file, but instead of the IP addresses of the Pis, enter the virtual IPv6 address when we set up keepalived.

## Sources

- https://www.youtube.com/watch?v=IFVYe3riDRA
- https://www.youtube.com/watch?v=hPfk0qd4xEY&t=675s
- https://www.reddit.com/r/pihole/comments/d5056q/tutorial_v2_how_to_run_2_pihole_servers_in_ha/?sort=new
