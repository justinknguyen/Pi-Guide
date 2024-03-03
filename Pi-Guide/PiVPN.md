# PiVPN

Turn the Pi into a VPN server. When connected to any public network, being able to VPN to the Pi at home provides you with security/privacy and all access to your home network (i.e., essentially connected to your home WiFi network while away from home).

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
  - [1. Connect From Your Computer](#1-connect-from-your-computer)
  - [2. Connect From Your Phone](#2-connect-from-your-phone)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Installation

1. Update and Upgrade:
   ```
   sudo apt update
   sudo apt full-upgrade
   ```
2. Install curl:
   ```
   sudo apt install curl -y
   ```
3. Install PiVPN:
   ```
   sudo curl -L https://install.pivpn.io | bash
   ```
4. Go through the install wizard and make sure you select WireGuard. If you have Pi-Hole, make sure you select "Yes" when it asks to use Pi-Hole's DNS server for the VPN.

## Configuration

1. In your router settings, port forward the port `51820` to your Pi's IP address.
2. Create your WireGuard profile:
   ```
   sudo pivpn add
   ```

### 1. Connect From Your Computer

Install WireGuard on your computer from https://www.wireguard.com/install/. Next, enter the following into your SSH terminal. Remember to replace the section below with the profile name you created:
```  
sudo nano /home/pi/configs/[PROFILENAME].conf
```
Copy everything in this config file and make the same .conf file on your Windows computer by pasting everything in it. Now open WireGuard and open this .conf that you just created. You can now connect to the VPN.

### 2. Connect From Your Phone

Install the WireGuard app. Next, enter the following into your SSH terminal:

```
pivpn -qr PROFILENAME
```

Then scan the QR code with your phone. You can now connect to the VPN.

## Testing

Once activating the VPN, you should have internet access. If you are on a public WiFi network, go to https://www.dnsleaktest.com/ and take note of the IP address. Next, activate the VPN and run the test again. You should now see your home network's public IP address.

I recommend setting up a dynamicDNS for your router so your public IP address doesn't change.

## Troubleshooting

If you are able to connect to WiFi but unable to access devices on the LAN, you need to disable "Block untunneled traffic" within Wireguard client settings if you are on Windows.

If on your phone, you need to manually edit the "Allowed IPs" to `0.0.0.0/1, 128.0.0.0/1, ::/1, 8000::/1`. This broke the WiFi connection later on, so I changed it back to `0.0.0.0/0, ::/0` and it worked again, including accessing devices on LAN. If it still does not work, you might want to try enabling “Exlude private IPs” within Wireguard app settings.

## Sources

- https://www.pivpn.io/
- https://pimylifeup.com/raspberry-pi-wireguard/
- https://blog.crankshafttech.com/2021/03/how-to-setup-pihole-pivpn-unbound.html?m=1
