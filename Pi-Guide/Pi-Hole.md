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

1. For IPv4: <br>
   Under "LAN" and "DHCP Server", enter the IPv4 address you took note of earlier under "DNS Server 1" then hit "Apply".
2. For IPv6: <br>
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

Optionally, you can also set Local DNS names under "DNS Records" in the side-menu. I recommend adding a domain name for your Raspberry Pi and your router, so you don't always have to enter the IP Address in the search bar.

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

If you want to use multiple DNS servers and have the second server only as backup for when the primary server is unresponsive, follow the steps below. These steps assume you have [Unbound](/Pi-Guide/Unbound.md) installed.

1. Select the two upstream providers you want in Pi-Hole's settings (e.g., Quad9 (filtered, DNSSEC) and Custom addresses for Unbound).
2. Create a file called `99-custom.conf` by entering the below command. In the file, type in `strict-order` and save.
   ```
   sudo nano /etc/dnsmasq.d/99-custom.conf
   ```
3. Ensure that the primary DNS server you want to use is listed first in the following file:
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
4. Restart the DNS:
   ```
   pihole restartdns
   ```

Pi-Hole has a feature that uses the fastest DNS resolver, so it's possible that it doesn't follow the order you set.

## Testing

Go to any site you know with ads and check if they're blocked. Make sure you turn off any ad-blocking extensions you may have. A site I recommend is https://www.speedtest.net/.

If you have IPv6 enabled, you can test if IPv6 is working by going to https://test-ipv6.com/, then making sure ad-block works.

## Troubleshooting

- If you're getting "Rate Limit" errors in Pi-Hole, perform the following (note: newer Pi-Hole versions have this setting built-in under the `Settings` window):

  1. Enter:
     ```
     sudo nano /etc/pihole/pihole-FTL.conf
     ```
  2. Type the following line in: `RATE_LIMIT=0/0`
     
     This will uncap the Rate Limit, however, it's better to simply raise the limit. I have mine at 2000/600. To find a limit tailored to you, login to Pi-Hole and hover over the highest bar under “Client activity over last 24 hours”. Take note of the highest number then add +25% to it. This number will be your first number, and 600 should be your second number representing 10 mins.

- If you have an Asus router and you suspect IPv6 is breaking Pi-Hole, perform the second half of the steps outlined here, [Getting IPv6 to Work with Unbound](/Pi-Guide/Unbound.md#getting-ipv6-to-work-with-unbound).
- If your ad-blocking does not work, try updating Pi-Hole with `pihole -up` or changing Interface settings to "Permit all origins". iCloud Private Relay must be turned off.
- If you changed your port for Pi-Hole, you might have to change it again if you update Pi-Hole with `pihole -up`. Follow the steps outlined at the top in the following link: [diyHue - Prerequisites](/Pi-Guide/diyHue.md#prerequisites)

## Sources

- https://www.youtube.com/watch?v=FnFtWsZ8IP0&t=851s
- https://github.com/anudeepND/whitelist
- https://firebog.net/
- https://discourse.pi-hole.net/t/using-redundant-upstream-dns-servers-w-cloudflared-lost-connection-to-api/18197
