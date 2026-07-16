# PiVPN

Turn the Pi into a VPN server, giving you security/privacy and full access to your home network from anywhere — as if you were connected to your home WiFi.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Connect From Your Computer](#1-connect-from-your-computer)
  - [2. Connect From Your Phone](#2-connect-from-your-phone)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Docker Alternative: wg-easy](#docker-alternative-wg-easy)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```bash
   sudo apt update
   sudo apt full-upgrade
   ```
1. Install curl and PiVPN:
   ```bash
   sudo apt install curl -y
   sudo curl -L https://install.pivpn.io | bash
   ```
1. Go through the install wizard and select WireGuard. If you have Pi-hole, select "Yes" when asked to use Pi-hole's DNS server for the VPN.

## Configuration

1. In your router settings, port forward `51820` to your Pi's IP address.
1. Create your WireGuard profile:
   ```bash
   sudo pivpn add
   ```

### 1. Connect From Your Computer

Install WireGuard on your computer from https://www.wireguard.com/install/. Then enter the following into your SSH terminal, replacing `[PROFILENAME]` with the profile name you created:
```bash
sudo nano /home/pi/configs/[PROFILENAME].conf
```
Copy the contents of this config file into a new `.conf` file on your computer. Open WireGuard, load that file, and connect.

### 2. Connect From Your Phone

Install the WireGuard app. Next, enter the following into your SSH terminal:

```bash
pivpn -qr [PROFILENAME]
```

Then scan the QR code with your phone. You can now connect to the VPN.

## Testing

Once you activate the VPN, you should still have internet access. To confirm it's working: on a public WiFi network, go to https://www.dnsleaktest.com/ and note the IP shown, then activate the VPN and run the test again — you should now see your home network's public IP address.

Setting up dynamic DNS is recommended so your VPN keeps working when your public IP address changes — see [DDNS](/Pi-Guide/DDNS.md).

## Troubleshooting

- No internet access: Run `pivpn -d`. If you see the prompt `[ERR] Iptables MASQUERADE rule is not set, attempt fix now? [Y/n]`, enter `y`.
- Able to connect to WiFi but unable to access devices on the LAN: Disable "Block untunneled traffic" within WireGuard client settings if you are on Windows.

## Docker Alternative: wg-easy

wg-easy runs the same WireGuard VPN as PiVPN, but adds a web UI where you create profiles and scan QR codes from a page instead of the command line. Run one or the other, not both — they both use UDP port 51820.

1. Have [Docker](/Pi-Guide/Docker.md) installed, then create a folder and download the official compose file:
   ```bash
   mkdir ~/wg-easy && cd ~/wg-easy
   curl -O https://raw.githubusercontent.com/wg-easy/wg-easy/master/docker-compose.yml
   ```
1. The admin UI expects HTTPS by default. To use it over plain HTTP on your home network, open the file and add an `INSECURE` environment variable to the `wg-easy` service (there's a commented example in the file):
   ```bash
   nano docker-compose.yml
   ```
   ```yaml
       environment:
         - INSECURE=true
   ```
1. Start it:
   ```bash
   docker compose up -d
   ```
1. In your router settings, port forward `51820` (UDP) to your Pi's IP address — same as PiVPN.
1. Open `http://[PIIPADDRESS]:51821` and follow the setup wizard: create your admin account, and set the host to the address clients connect to from outside (your [DDNS](/Pi-Guide/DDNS.md) hostname, or your public IP).
1. Add a client in the UI, then scan its QR code with the WireGuard app on your phone, or download the `.conf` file for your computer.
   - To get [Pi-hole](/Pi-Guide/Pi-hole.md) ad-blocking over the VPN, set the client's DNS to the Pi's IP address in the client's settings before downloading it.

The same [Testing](#testing) steps above apply.

## Sources

- https://www.pivpn.io/
- https://pimylifeup.com/raspberry-pi-wireguard/
- https://blog.crankshafttech.com/2021/03/how-to-setup-pihole-pivpn-unbound.html?m=1
- https://wg-easy.github.io/wg-easy/latest/
