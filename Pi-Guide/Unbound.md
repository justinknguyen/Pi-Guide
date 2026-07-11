# Unbound

Recursive DNS for Pi-Hole. Resolves faster than iterative queries and adds privacy by cutting out third parties like Google, Cloudflare, and OpenDNS.

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Getting IPv6 to Work with Unbound](#getting-ipv6-to-work-with-unbound)
- [Sources](#sources)

## Prerequisites

[Pi-Hole](/Pi-Guide/Pi-Hole.md)

## Installation

1. Update and install Unbound:
   ```bash
   sudo apt update
   sudo apt install unbound
   ```
1. Download the current root hints file:
   ```bash
   wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
   ```

## Configuration

1. In your Pi-Hole web settings, go to `Settings > DNS` then scroll down to `Advanced DNS settings` and turn off "Use DNSSEC"
1. Pi-hole v6 ignores `/etc/dnsmasq.d/*.conf` files by default, so enable that first:
   ```bash
   sudo pihole-FTL --config misc.etc_dnsmasq_d true
   ```
   (or via the web UI: `Settings > All Settings > Miscellaneous > etc_dnsmasq_d`)
1. Set `cache-size=0` in the following file:
   ```bash
   sudo nano /etc/dnsmasq.d/01-pihole.conf
   ```
1. Add the following to `sudo nano /etc/unbound/unbound.conf`:
   ```
   server:
       # These options should be added to the existing server configuration,
       # overwriting existing values if they're there.

       # This refreshes expiring cache entries if they have been accessed with
       # less than 10% of their TTL remaining
       prefetch: yes

       # This attempts to reduce latency by serving the outdated record before
       # updating it instead of the other way around. Alternative is to increase
       # cache-min-ttl to e.g. 3600.
       cache-min-ttl: 0
       serve-expired: yes

       # Larger caches than the 4m/4m defaults — overkill for a small network,
       # but the RAM would otherwise sit idle. Keep rrset about 2x msg.
       msg-cache-size: 128m
       rrset-cache-size: 256m
   ```
1. Create the unbound config file for Pi-Hole:
   ```bash
   sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
   ```
1. Paste the following in.

   <ins>IMPORTANT:</ins>
   - If you have IPv6 for your network, change `do-ip6: no` to `do-ip6: yes` below.
   - If you're installing Unbound again for another Pi, change the port to `5353` to avoid a conflict.

   ```
   server:
       # If no logfile is specified, syslog is used
       # logfile: "/var/log/unbound/unbound.log"
       verbosity: 0

       interface: 127.0.0.1
       port: 5335
       do-ip4: yes
       do-udp: yes
       do-tcp: yes

       # May be set to yes if you have IPv6 connectivity
       do-ip6: no

       # You want to leave this to no unless you have *native* IPv6. With 6to4 and
       # Terredo tunnels your web browser should favor IPv4 for the same reasons
       prefer-ip6: no

       # Use this only when you downloaded the list of primary root servers!
       # This guide downloads it in the Installation section above, so it's enabled here.
       # If you skipped that step and use the default dns-root-data package instead,
       # comment this line out — unbound will find the root servers automatically.
       root-hints: "/var/lib/unbound/root.hints"

       # Trust glue only if it is within the server's authority
       harden-glue: yes

       # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
       harden-dnssec-stripped: yes

       # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
       # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
       use-caps-for-id: no

       # Reduce EDNS reassembly buffer size to avoid unreliable IP fragmentation
       # over UDP (recommended by DNS Flag Day 2020)
       edns-buffer-size: 1232

       # Perform prefetching of close to expired message cache entries
       # This only applies to domains that have been frequently queried
       prefetch: yes

       # One thread is sufficient for a small home network
       num-threads: 1

       # Ensure kernel buffer is large enough to not lose messages in traffic spikes
       so-rcvbuf: 1m

       # Ensure privacy of local IP ranges
       private-address: 192.168.0.0/16
       private-address: 169.254.0.0/16
       private-address: 172.16.0.0/12
       private-address: 10.0.0.0/8
       private-address: fd00::/8
       private-address: fe80::/10
   ```
1. To save the file, press `Ctrl+X` then `Y` then `Enter`.
1. Restart Unbound:
   ```bash
   sudo service unbound restart
   ```
1. Go to the WebUI for Pi-Hole and head to "Settings" then "DNS", and uncheck whatever is checked under "Upstream DNS Servers".
1. Under "Custom 1 (IPv4)" enter:
   ```
   127.0.0.1#5335
   ```
1. If you have IPv6, under "Custom 3 (IPv6)" enter:
   ```
   ::1#5335
   ```

## Testing

1. Query the following:
   ```bash
   dig pi-hole.net @127.0.0.1 -p 5335
   ```
1. Look for `status: NOERROR` and an answer section in the output:

   ```
   ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25728
   ...
   ;; ANSWER SECTION:
   pi-hole.net.            300     IN      A       3.18.136.52
   ```

   If you have IPv6 enabled, it might fail with `SERVFAIL` instead — the fix is in [Getting IPv6 to Work with Unbound](#getting-ipv6-to-work-with-unbound) below.

1. You can test DNSSEC validation using the commands below. The first command should give a status report of `SERVFAIL`, and the second should give `NOERROR`.
   ```bash
   dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
   dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
   ```

## Getting IPv6 to Work with Unbound

IPv6 can be tricky to get working with Unbound, at least on an Asus router. If IPv6 is breaking Pi-Hole and Unbound, do the following:

1. Edit file `resolvconf.conf` and comment out the last line which should read, `unbound_conf=/etc/unbound/unbound.conf.d/resolvconf_resolvers.conf`.
   ```bash
   sudo nano /etc/resolvconf.conf
   ```
1. Delete the unwanted unbound configuration file and restart unbound:
   ```bash
   sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
   sudo service unbound restart
   ```

### This MAY Help

If the above still doesn't work, the following applies to Asus routers. If you have keepalived installed, use the IPv6 vrrp address instead of your two Pis' addresses.

On an Asus router, first go to "Administration" then "System" and enable SSH, SSH Port Forwarding, and "Enable JFFS custom scripts and configs".

1. SSH into your router and enter:
   ```bash
   nano /jffs/scripts/dnsmasq.postconf
   ```
1. Paste the following in. Make sure you enter your IPv6 address within the square brackets below. You can get rid of
   `,[IPv6 address of second Pi]` if you don't have a second Pi.
   ```bash
   #!/bin/sh
   CONFIG=$1
   source /usr/sbin/helper.sh
   pc_replace "dhcp-option=lan,option6:23,[::]" "dhcp-option=lan,option6:23,[IPv6 address of first Pi],[IPv6 address of second Pi]" $CONFIG
   ```
1. Enter:
   ```bash
   chmod 755 /jffs/scripts/dnsmasq.postconf
   ```
1. Reboot the router.
1. SSH into Pi and enter:
   ```bash
   sudo nano /etc/dhcpcd.conf
   ```
1. Scroll down to the bottom and you should set the lines similar to below.
   ```
   interface eth0
       static ip_address= [IPv4 Address of the Pihole]/24
       static ip6_address= [IPv6 Address of the Pihole]/64
       static routers=[IP Address of the router]
       static domain_name_servers=[IP Address of the router] [LAN IPv6 Address of the router]
   ```
1. Restart services:
   ```bash
   sudo service pihole-FTL restart
   sudo systemctl restart dhcpcd
   sudo service unbound restart
   ```

## Sources

- https://www.youtube.com/watch?v=FnFtWsZ8IP0&t=851s
- https://docs.pi-hole.net/guides/dns/unbound/
- https://www.snbforums.com/threads/asuswrt-merlin-serving-ipv6-router-ip-instead-of-ipv6-dns-server-ip-f-w-384-19.67225/
- https://discourse.pi-hole.net/t/the-client-pi-hole-triggers-a-warning-in-dnsmasq-core-that-the-maximum-number-of-concurrent-dns-requests-has-been-reached/52475/3
- https://www.reddit.com/r/pihole/comments/d9j1z6/unbound_as_recursive_dns_server_slow_performance/?share_id=U6aobkq-7q_O9nuWmA6Am&utm_content=2&utm_medium=ios_app&utm_name=ioscss&utm_source=share&utm_term=1
