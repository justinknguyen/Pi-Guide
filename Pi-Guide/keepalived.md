# keepalived

keepalived creates a High Availability setup between two Pis — one acts as the master server, the other as backup. If the master goes down, traffic redirects to the backup until the master comes back online.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- [Pi-hole](/Pi-Guide/Pi-hole.md)
- [Unbound](/Pi-Guide/Unbound.md) (recommended)
- [nebula-sync](/Pi-Guide/Nebula-Sync.md) (recommended) — keeps the two Pi-holes' settings in sync

## Installation

1.  On both Pis, update, install keepalived and the ipset runtime library (package name depends on OS version — `libipset13` on Raspberry Pi OS "Bookworm"/Debian 12, `libipset13t64` on "Trixie"/Debian 13+), then enable the keepalived service:
    ```bash
    sudo apt update
    sudo apt install keepalived
    sudo apt install libipset13 || sudo apt install libipset13t64
    sudo systemctl enable keepalived.service
    ```
1.  Create the pihole-FTL service check script. Run the following commands on both Pis:
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
1.  Now add the keepalived configuration on the primary Pi-hole machine (the master/active server). Adjust the settings for your setup.
    ```bash
    sudo nano /etc/keepalived/keepalived.conf
    ```
1. Copy and paste the following in.

    <ins>IMPORTANT:</ins> The password you enter below has a maximum of 8 characters.

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

| Option | Description |
| --- | --- |
| `router_id` | A unique name, for instance the Pi-hole hostname. |
| `state` | Master/Active or Backup/Standby server. |
| `interface` | The network interface that Pi uses — `eth0` for ethernet, `wlan0` for WiFi (the example configs here use `eth0` on the master and `wlan0` on the backup; set each to match your setup). |
| `virtual_router_id` | Any number between 0 and 255. Must match on the Master and Backup configs. |
| `priority` | The master server should have a higher priority than the backup server. |
| `unicast_src_ip` | The IP address of this (source) server. |
| `unicast_peer` | The IP address of the other (peer) server. |
| `auth_pass` | A password (max 8 characters). Must match on the Master and Backup configs. |
| `virtual_ipaddress` | The High Availability IP address. |

1. If you have IPv6 enabled on Pi-hole, enter the following after everything above. Double-check the IPv6 addresses with `ifconfig` first.

    <ins>IMPORTANT:</ins> The password you enter below has a maximum of 8 characters.

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

1. Now add the keepalived configuration on the secondary Pi-hole machine (the Backup/Standby server).
    ```bash
    sudo nano /etc/keepalived/keepalived.conf
    ```
1. Copy and paste the following in.

    <ins>IMPORTANT:</ins> The password you enter below has a maximum of 8 characters.

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

1. If you have IPv6 enabled on Pi-hole, enter the following after everything above. Double-check the IPv6 addresses with `ifconfig` first.

    <ins>IMPORTANT:</ins> The password you enter below has a maximum of 8 characters.

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

1. Head into your router settings and replace the DNS servers for Pi-hole with the virtual IP address you configured with keepalived.
1. Set a crontask to restart the keepalived service after a reboot. Enter on both Pis:
   ```bash
   sudo crontab -e
   ```
1. Copy and paste the following in at the bottom:
   ```bash
   @reboot sleep 60 && sudo systemctl restart keepalived.service
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. On both Pis, set a delay for when Pi-hole starts up after a reboot. Pi-hole v6 replaced `/etc/pihole/pihole-FTL.conf` with `/etc/pihole/pihole.toml`, so set this via the CLI instead of editing a conf file directly:
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
1. For example, there may be a noticeable time difference between the ping times if one Pi is connected by ethernet while the other is not. This makes it easy to see the switching between the two Pis in real time. If the difference between the two Pis isn't noticeable this way, manually check the logs of each by entering `sudo systemctl status keepalived.service`. Start by pinging the virtual IP address constantly by entering:
   ```bash
   ping [VIRTUALIPADDRESS] -t
   ```
1. Now, `sudo reboot` the primary Pi. The ping time should jump up randomly, indicating the virtual IP address switched to the secondary Pi. Once the primary Pi boots back up, the ping time should lower and become consistent again. If the virtual IP address does not switch back to the primary Pi, follow the Troubleshooting steps below.
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
- If it happens again, it could be an IPv6 issue:
    - double check both of your Pis' IPv6 addresses with `ifconfig`. If it changed, update it inside of the keepalived config file.
    - try disabling and re-enabling router advertisement for IPv6 in router settings.
    - one reason is because of a file in your router when setting up [Unbound](/Pi-Guide/Unbound.md#this-may-help). The fix would be to keep the file, but instead of the IP addresses of the Pis, enter the virtual IPv6 address configured earlier in the keepalived setup.

## Sources

- https://www.youtube.com/watch?v=IFVYe3riDRA
- https://www.youtube.com/watch?v=hPfk0qd4xEY&t=675s
- https://www.reddit.com/r/pihole/comments/d5056q/tutorial_v2_how_to_run_2_pihole_servers_in_ha/?sort=new
