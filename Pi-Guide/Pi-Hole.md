# Pi-Hole

Network-wide ad-blocking.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Router Settings](#1-router-settings)
  - [2. PiHole DNS Settings](#2-pi-hole-dns-settings)
  - [3. Adding Adlists](#3-adding-adlists)
  - [4. Adding Whitelists](#4-adding-whitelists)
  - [5. Multiple Upstream DNS Servers (Optional)](#5-multiple-upstream-dns-servers-optional)
- [Testing](#testing)
- [Updating](#updating)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Installation

1. Install Pi-Hole:
   ```
   sudo curl -sSL https://install.pi-hole.net | bash
   ```
2. Go through the install wizard using default settings (just keep pressing Enter/Yes).
3. Once installed, take note of the IPv4 and (if enabled) IPv6 address. This will be used in your router settings.
4. Change the Pi-Hole login password by entering:
   ```
   pihole -a -p [NEWPASSWORD]
   ```

## Configuration

### 1. Router Settings

Using an Asus router,

1. Under "WAN" and "WAN DNS Setting", ensure "Connect to DNS Server automatically" is set to `Yes`.
2. For IPv4: <br>
   Under "LAN" and "DHCP Server", enter the IPv4 address you took note of earlier under "DNS Server 1" and disable Router advertisement, then hit "Apply".
3. For IPv6: <br>
   Under "IPv6", enter the IPv6 address you took note of earlier under "IPv6 DNS Server 1", then hit "Apply".

### 2. Pi-Hole DNS Settings

Login to Pi-Hole by typing `[PIIPADDRESS]/admin` into your search bar. Head to "Settings" then "DNS". Here you'll see the upstream DNS server you're using. I recommend using "Quad9 (filtered, DNSSEC)". Ensure you check both boxes under the "IPv4" column. Same applies to IPv6 if you have it enabled.

For `Interface settings`, I have "Allow only local requests" checked, but if you notice any devices not being ad-blocked, select "Permit all origins".

For `Advanced DNS settings`, I enabled the first two check boxes, set the rate-limiting to 2000 and 600, and also enabled conditional forwarding. Conditional forwarding allows me to view the name of devices in the client list of Pi-Hole. Depending on your router, your IP address will look a little different, but it should be similar to something like this:

- Local network in CIDR notation: 192.168.50.0/24
  - the format is your router's IP address but with a 0 as the last digit, then add /24.
- IP address of your DHCP server (router): 192.168.50.1
  - the format is just your router's IP address.
- Local domain name (optional): router.asus.com
  - the format is the domain you use to sign into your router's settings.

### 3. Adding Adlists

Click on "Group Management" then "Adlists" and add any adlist you want. I recommend https://raw.githubusercontent.com/hagezi/dns-blocklists/main/adblock/pro.txt from https://github.com/hagezi/dns-blocklists. You can copy and paste multiple links at a time.

Once added, either enter `pihole -g` into PuTTY or within the WebUI, go to "Tools" then "Update Gravity". Finally, click on "Update".

### 4. Adding Whitelists

1. Install python3:
   ```
   sudo apt install python3
   ```
2. Install whitelist:
   ```
   git clone https://github.com/anudeepND/whitelist.git
   sudo python3 whitelist/scripts/whitelist.py
   ```
   An important whitelist you need to add manually within the WebUI is `codeload.github.com`. This is to prevent future program installs from being blocked.

### 5. Multiple Upstream DNS Servers (Optional)

If you want to use multiple DNS servers and have the second server only as backup for when the primary server is unresponsive, follow the steps below. These steps assume you have [Unbound](/Pi-Guide/Unbound.md), [Gravity Sync](/Pi-Guide/Gravity%20Sync.md), and [keepalived](/Pi-Guide/keepalived.md) installed.

1. Select the two upstream providers you want in Pi-Hole's settings (e.g., Quad9 (filtered, DNSSEC) and Custom addresses for Unbound).
2. Create a file called `99-custom.conf` by entering the below command. In the file, type in `strict-order` and save.
   ```
   sudo nano /etc/dnsmasq.d/99-custom.conf
   ```
3. Ensure that the primary DNS server you want to use is listed first in the following file:
   ```
   sudo nano /etc/dnsmasq.d/01-pihole.conf
   ```
   - e.g., The first two address are Custom addresses for Unbound, and the bottom four are for Quad9:
     ```
     server=127.0.0.1#5335
     server=::1#5335
     server=9.9.9.9
     server=149.112.112.112
     server=2620:fe::fe
     server=2620:fe::9
     ```
4. Do the same thing in the following file:
   ```
   sudo nano /etc/pihole/setupVars.conf
   ```
   - e.g., The first two address are Custom addresses for Unbound, and the bottom four are for Quad9:
     ```
     PIHOLE_DNS_1=127.0.0.1#5335
     PIHOLE_DNS_2=::1#5335
     PIHOLE_DNS_3=9.9.9.9
     PIHOLE_DNS_4=149.112.112.112
     PIHOLE_DNS_5=2620:fe::fe
     PIHOLE_DNS_6=2620:fe::9
     ```
5. Restart the DNS:
   ```
   pihole restartdns
   ```
   Having multiple upstream servers might result in odd behaviour if you were to restart the Pi or update Pi-Hole using `pihole -up`.

- Upon restarting,
  - Pi-Hole might create a new config file named `01-pihole.conf.save` (appends .save to the end) and DNS will not start. This happened on my Pi 4, but did not on my Pi Zero 2 W. Check if you have it by entering the following:
    ```
    cd /etc/dnsmasq.d/
    ls
    ```
  - To fix this, try deleting the `.conf.save` file and see if it creates it again.
    ```
    sudo rm /etc/dnsmasq.d/01-pihole.conf
    ```
  - Otherwise, delete everything within the original `01-pihole.conf` file and it will work from now on. Restarting the Pi should no longer create more config files.
- Upon updating,
  - If you have a `01-pihole.conf.save` file, update might fail because it repopulates `01-pihole.conf` again.
  - To fix this, try deleting the `.conf.save` file and see if it creates it again.
    ```
    sudo rm /etc/dnsmasq.d/01-pihole.conf
    ```
  - Otherwise, delete everything within the original `01-pihole.conf` file and attempt to update (note: successful update may rearrange/repopulate `01-pihole.conf` once again).

## Testing

Go to any site you know with ads and check if they're blocked. Make sure you turn off any ad-blocking extensions you may have. A site I recommend is https://www.speedtest.net/.

If you have IPv6 enabled, you can test if IPv6 is working by going to https://test-ipv6.com/, then making sure ad-block works.

## Updating

When updating with `pihole -up` there are a couple of things to check if you have multiple upstream DNS servers or you changed your Pi-Hole port.

1. If you have multiple upstream DNS servers:
   - If you have a `01-pihole.conf.save` file, update might fail because it repopulates `01-pihole.conf` again.
   - To fix this, try deleting the `.conf.save` file and see if it creates it again.
     ```
     sudo rm /etc/dnsmasq.d/01-pihole.conf
     ```
   - Otherwise, delete everything within the original `01-pihole.conf` file and attempt to update (note: successful update may rearrange/repopulate `01-pihole.conf` once again).
2. If you changed your port for Pi-Hole:
   - You might have to change it again by following the steps outlined at the top in the following link: [diyHue - Prerequisites](/Pi-Guide/diyHue.md#prerequisites)

## Troubleshooting

- If you're getting "Rate Limit" errors in Pi-Hole, perform the following (note: newer Pi-Hole versions have this setting built-in under the `Settings` window):

  1. Enter:
     ```
     sudo nano /etc/pihole/pihole-FTL.conf
     ```
  2. Type the following line in:
     `   RATE_LIMIT=0/0`
     This will uncap the Rate Limit, however, it's better to simply raise the limit. I have mine at 2000/600. To find a limit tailored to you, login to Pi-Hole and hover over the highest bar under “Client activity over last 24 hours”. Take note of the highest number then add +25% to it. This number will be your first number, and 600 should be your second number representing 10 mins.

- If you have an Asus router and you suspect IPv6 is breaking Pi-Hole, perform the second half of the steps outlined here, [Getting IPv6 to Work with Unbound](/Pi-Guide/Unbound.md#getting-ipv6-to-work-with-unbound).
- If your ad-blocking does not work in the future, try updating Pi-Hole with `pihole -up` or changing Interface settings to "Permit all origins".
- If you changed your port for Pi-Hole, you might have to change it again if you update Pi-Hole with `pihole -up`. Follow the steps outlined at the top in the following link: [diyHue - Prerequisites](/Pi-Guide/diyHue.md#prerequisites)

## Sources

- https://www.youtube.com/watch?v=FnFtWsZ8IP0&t=851s
- https://github.com/anudeepND/whitelist
- https://firebog.net/
- https://discourse.pi-hole.net/t/using-redundant-upstream-dns-servers-w-cloudflared-lost-connection-to-api/18197
