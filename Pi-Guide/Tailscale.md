# Tailscale

Zero-config VPN (built on WireGuard) that connects your devices into a private network wherever they are. Reach your Pi from anywhere with **no port forwarding** — it works even behind CGNAT where [PiVPN](/Pi-Guide/PiVPN.md) can't. Free for personal use (up to 100 devices).

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Access Your Whole Home Network (Subnet Router)](#1-access-your-whole-home-network-subnet-router)
  - [2. Route All Traffic Through Home (Exit Node)](#2-route-all-traffic-through-home-exit-node)
  - [3. Pi-hole Ad-Blocking Everywhere](#3-pi-hole-ad-blocking-everywhere)
- [Testing](#testing)
- [Sources](#sources)

## Installation

1. Install Tailscale:
   ```bash
   curl -fsSL https://tailscale.com/install.sh | sh
   ```
1. Bring it up and sign in with the link it prints:
   ```bash
   sudo tailscale up
   ```
1. In the admin console (https://login.tailscale.com/admin/machines), click the "…" menu on the Pi and select "Disable key expiry" so you never have to re-authenticate it.
1. Install the Tailscale app on your phone/laptop and sign in to the same account. That's it — every device can now reach the Pi at its `100.x.y.z` Tailscale IP or its hostname.

## Configuration

The steps below are optional but are most of the value of running Tailscale on a Pi.

### 1. Access Your Whole Home Network (Subnet Router)

Lets your remote devices reach *everything* on your home LAN (router, printers, other Pis), not just this Pi.

1. Enable IP forwarding:
   ```bash
   echo 'net.ipv4.ip_forward = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
   echo 'net.ipv6.conf.all.forwarding = 1' | sudo tee -a /etc/sysctl.d/99-tailscale.conf
   sudo sysctl -p /etc/sysctl.d/99-tailscale.conf
   ```
1. Advertise your LAN subnet (replace with yours — it's your router's IP with a `0` last digit, plus `/24`):
   ```bash
   sudo tailscale up --advertise-routes=192.168.50.0/24
   ```
1. In the [admin console](https://login.tailscale.com/admin/machines), click on the Pi and find the "Subnets" tag next to its name — click it to open "Edit route settings," then check the box next to `192.168.50.0/24` and save. <ins>This step is required</ins> — advertising a route with `--advertise-routes` does not activate it. The tag showing up next to the Pi just means it's been advertised, not approved; until you check this box, remote devices will connect to Tailscale fine but silently fail to reach anything on your LAN.

### 2. Route All Traffic Through Home (Exit Node)

Sends *all* your device's traffic through your home connection (like a traditional VPN) — useful on public WiFi.

```bash
sudo tailscale up --advertise-routes=192.168.50.0/24 --advertise-exit-node
```

Same as above: open "Edit route settings" for the Pi in the admin console and check "Use as exit node," then save — advertising it alone isn't enough. Once approved, select the Pi as the exit node in the Tailscale app on your device when you want it.

### 3. Pi-hole Ad-Blocking Everywhere

If this Pi runs [Pi-hole](/Pi-Guide/Pi-hole.md), you can get ad-blocking on your phone wherever you are:

1. In the [Tailscale admin console](https://login.tailscale.com/admin/dns) (not Pi-hole) — go to the DNS tab.
1. Still in the Tailscale admin console: add the Pi's Tailscale IP (`100.x.y.z`, shown with `tailscale ip -4`) as a global nameserver and enable "Override local DNS". This tells every device on your Tailnet to send DNS queries to the Pi instead of their normal DNS, even away from home.
1. Now in Pi-hole's web UI, make sure Interface settings allow answering on the Tailscale interface ("Permit all origins" is the simple option) — otherwise Pi-hole ignores the queries Tailscale just started sending it.

## Testing

Turn off WiFi on your phone (use cellular), enable Tailscale, and open `http://[TAILSCALEIP]` for any service on the Pi — e.g. Pi-hole's admin page. With the exit node selected, https://www.dnsleaktest.com/ should show your home IP.

## Sources

- https://tailscale.com/kb/1174/install-debian-bookworm
- https://tailscale.com/kb/1019/subnets
- https://tailscale.com/kb/1103/exit-nodes
- https://tailscale.com/kb/1114/pi-hole
