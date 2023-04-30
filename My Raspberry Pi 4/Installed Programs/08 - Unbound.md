# Unbound 
Recursive DNS for Pi-Hole. Tends to resolve faster than iterative queries and also provides privacy by getting rid of the third party, such as Google, Cloudflare, OpenDNS, etc.
## Installation
1. Update:
    ```
    sudo apt update
    ```
2. Install Unbound:
    ```
    sudo apt install unbound
    ```
3. Download the current root hints file:
    ```
    wget https://www.internic.net/domain/named.root -qO- | sudo tee /var/lib/unbound/root.hints
    ```
## Configuration
1. Create the unbound config file:
    ```
    sudo nano /etc/unbound/unbound.conf.d/pi-hole.conf
    ```
2. Paste the following in (IMPORTANT: if you have IPv6 for your network change line below to `do-ip6: yes`. Also, if you’re configuring this for a second Pi, you should change the port to `5353` to avoid conflict.):
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
        # If you use the default dns-root-data package, unbound will find it automatically
        #root-hints: "/var/lib/unbound/root.hints"

        # Trust glue only if it is within the server's authority
        harden-glue: yes

        # Require DNSSEC data for trust-anchored zones, if such data is absent, the zone becomes BOGUS
        harden-dnssec-stripped: yes

        # Don't use Capitalization randomization as it known to cause DNSSEC issues sometimes
        # see https://discourse.pi-hole.net/t/unbound-stubby-or-dnscrypt-proxy/9378 for further details
        use-caps-for-id: no

        # Reduce EDNS reassembly buffer size.
        # IP fragmentation is unreliable on the Internet today, and can cause
        # transmission failures when large DNS messages are sent via UDP. Even
        # when fragmentation does work, it may not be secure; it is theoretically
        # possible to spoof parts of a fragmented DNS message, without easy
        # detection at the receiving end. Recently, there was an excellent study
        # >>> Defragmenting DNS - Determining the optimal maximum UDP response size for DNS <<<
        # by Axel Koolhaas, and Tjeerd Slokker (https://indico.dns-oarc.net/event/36/contributions/776/)
        # in collaboration with NLnet Labs explored DNS using real world data from the
        # the RIPE Atlas probes and the researchers suggested different values for
        # IPv4 and IPv6 and in different scenarios. They advise that servers should
        # be configured to limit DNS messages sent over UDP to a size that will not
        # trigger fragmentation on typical network links. DNS servers can switch
        # from UDP to TCP when a DNS response is too big to fit in this limited
        # buffer size. This value has also been suggested in DNS Flag Day 2020.
        edns-buffer-size: 1232

        # Perform prefetching of close to expired message cache entries
        # This only applies to domains that have been frequently queried
        prefetch: yes

        # One thread should be sufficient, can be increased on beefy machines. In reality for most users running on small networks or on a single machine, it should be unnecessary to seek performance enhancement by increasing num-threads above 1.
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
3. To save the file, press `Ctrl+X` then `Y` then `Enter`.
4. Restart Unbound:
    ```
    sudo service unbound restart
    ```
5. Go to the WebUI for Pi-Hole and head to "Settings" then "DNS", and uncheck whatever is checked under "Upstream DNS Servers".
6. Under "Custom 1 (IPv4)" enter:
    ```
    127.0.0.1#5335
    ```
7. If you have IPv6, under "Custom 3 (IPv6)" enter:
    ```
    ::1#5335
    ```
## Testing
1. Query the following:
    ```
    dig pi-hole.net @127.0.0.1 -p 5335
    ```
2. You should see something similar below (look for `status: NOERROR`). If you have IPv6 enabled, it's likely this will fail and you'll get `SERVFAIL`. The solution to fix that is provided in the next section.
    ```
    pi@pi4:~ $ dig pi-hole.net @127.0.0.1 -p 5335

    ; <<>> DiG 9.16.22-Raspbian <<>> pi-hole.net @127.0.0.1 -p 5335
    ;; global options: +cmd
    ;; Got answer:
    ;; ->>HEADER<<- opcode: QUERY, status: NOERROR, id: 25728
    ;; flags: qr rd ra; QUERY: 1, ANSWER: 1, AUTHORITY: 0, ADDITIONAL: 1

    ;; OPT PSEUDOSECTION:
    ; EDNS: version: 0, flags:; udp: 1232
    ;; QUESTION SECTION:
    ;pi-hole.net.                   IN      A

    ;; ANSWER SECTION:
    pi-hole.net.            300     IN      A       3.18.136.52

    ;; Query time: 59 msec
    ;; SERVER: 127.0.0.1#5335(127.0.0.1)
    ;; WHEN: Sat Feb 12 17:42:12 MST 2022
    ;; MSG SIZE  rcvd: 56
    ```
3. You can test DNSSEC validation using the commands below. The first command should give a status report of `SERVFAIL` and no IP address. The second should give `NOERROR` plus an IP address.
    ```
    dig sigfail.verteiltesysteme.net @127.0.0.1 -p 5335
    dig sigok.verteiltesysteme.net @127.0.0.1 -p 5335
    ```
## Getting IPv6 to Work with Unbound
IPv6 is tricky to get working with Unbound, with an Asus router at least. If your IPv6 is breaking Pi-Hole and Unbound, then perform the following:
1. Edit file `resolvconf.conf` and comment out the last line which should read, `unbound_conf=/etc/unbound/unbound.conf.d/resolvconf_resolvers.conf`.
    ```
    sudo nano /etc/resolvconf.conf
    ```
2. Delete the unwanted unbound configuration file:
    ```
    sudo rm /etc/unbound/unbound.conf.d/resolvconf_resolvers.conf
    ```
3. Restart unbound:
    ```
    sudo service unbound restart
    ```
### This MAY Help
If the above steps still do not work, the below can be performed on an Asus router, however, if you have keepalived installed, you will want to put in the IPv6 vrrp address instead of your two Pi’s addresses.
1. SSH into your router (for an Asus router, enable SSH and SSH Port Forwarding by going to "Administration" then "System". Set "Enable JFFS custom scripts and configs" also.) and enter:
    ```
    nano /jffs/scripts/dnsmasq.postconf 
    ```
2. Paste the following in. Make sure you enter your IPv6 address within the square brackets below. You can get rid of
`,[IPv6 address of second Pi]` if you don't have a second Pi.
    ```
    #!/bin/sh
    CONFIG=$1
    source /usr/sbin/helper.sh
    pc_replace "dhcp-option=lan,option6:23,[::]" "dhcp-option=lan,option6:23,[IPv6 address of first Pi],[IPv6 address of second Pi]" $CONFIG
    ```
3. Enter:
    ```
    chmod 755 /jffs/scripts/dnsmasq.postconf
    ```
4. Reboot the router.
5. SSH into Pi and enter:
    ```
    sudo nano /etc/dhcpcd.conf
    ```
6. Scroll down to the bottom and you should set the lines similar to below.
    ```
    interface eth0
        static ip_address= [IPv4 Address of the Pihole]/24
        static ip6_address= [IPv6 Address of the Pihole]/64
        static routers=[IP Address of the router]
        static domain_name_servers=[IP Address of the router] [LAN IPv6 Address of the router]
    ```
7. Restart services:
    ```
    sudo service pihole-FTL restart 
    sudo systemctl restart dhcpcd 
    sudo service unbound restart
    ``` 
## Sources
* https://www.youtube.com/watch?v=FnFtWsZ8IP0&t=851s
* https://docs.pi-hole.net/guides/dns/unbound/
* https://www.snbforums.com/threads/asuswrt-merlin-serving-ipv6-router-ip-instead-of-ipv6-dns-server-ip-f-w-384-19.67225/
* https://discourse.pi-hole.net/t/the-client-pi-hole-triggers-a-warning-in-dnsmasq-core-that-the-maximum-number-of-concurrent-dns-requests-has-been-reached/52475/3