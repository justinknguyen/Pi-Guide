# System Tuning & Cleanup

A checkup for a headless Pi that's been running for a while: get back space on the SD card, cut unnecessary writes to it, turn off things you don't use, and keep the firmware current. Each step is small, safe and easy to undo, and none of them is required. Pick the ones that fit your setup.

## Table of Contents

- [Find What's Using the SD Card](#find-whats-using-the-sd-card)
- [Remove Old Kernels and Cached Packages](#remove-old-kernels-and-cached-packages)
- [Swap Less](#swap-less)
- [Disable Services You Don't Use](#disable-services-you-dont-use)
- [Turn Off WiFi on a Wired Pi](#turn-off-wifi-on-a-wired-pi)
- [Update the Bootloader](#update-the-bootloader)
- [Check Temperature and Power](#check-temperature-and-power)
- [Sources](#sources)

## Find What's Using the SD Card

```bash
df -h /
sudo du -xh --max-depth=2 / 2>/dev/null | sort -h | tail -20
```

`-x` stays on the SD card and skips mounted drives. The usual suspects:

- **`/var/lib/docker`**: old container images pile up every time a container updates. This is often tens of GB. See [Docker: Keeping Disk Usage in Check](/Pi-Guide/Docker.md#keeping-disk-usage-in-check), and run [Watchtower](/Pi-Guide/Watchtower.md) with `WATCHTOWER_CLEANUP=true` so it doesn't come back.
- **Container logs**, also under `/var/lib/docker`. See [Docker: Limit Container Log Size](/Pi-Guide/Docker.md#limit-container-log-size).
- **`/usr/lib/modules` and `/usr/src`**: old kernels and headers (next section).
- **`~/.cache/pip`** and similar caches in your home folder, which are safe to delete: `pip cache purge`.

Anything big that's written to often (media, databases, backups) belongs on a USB drive, not the SD card. See [NAS](/Pi-Guide/NAS.md#configuration) for mounting one.

## Remove Old Kernels and Cached Packages

Each kernel update leaves the previous one installed, along with its headers if you have them. Over a year or two this adds up to several GB.

```bash
sudo apt autoremove --purge
sudo apt clean
```

`autoremove` keeps the running kernel and the newest one, so it's safe. Read its list before answering `Y` anyway. To keep the download cache from growing again, see [Unattended-Upgrades](/Pi-Guide/Unattended-Upgrades.md#2-keep-the-download-cache-from-growing).

## Swap Less

Raspberry Pi OS swaps to the SD card, and with the default `vm.swappiness=60` the kernel starts swapping well before memory is actually full. That's extra wear for little benefit on a Pi with 4 GB or more. Lowering it keeps swap as insurance against running out of memory, without routine use:

```bash
echo 'vm.swappiness=10' | sudo tee /etc/sysctl.d/99-swappiness.conf
sudo sysctl --system
cat /proc/sys/vm/swappiness   # should print 10
```

Undo: `sudo rm /etc/sysctl.d/99-swappiness.conf` and reboot.

Together with [Log2Ram](/Pi-Guide/Log2RAM.md), this removes most of the everyday writes to the card.

## Disable Services You Don't Use

Raspberry Pi OS enables a few services that a headless server rarely needs. Each one uses a little memory and, more importantly, is one more thing listening or running for no reason.

| Service | What it's for | Leave it on if... |
| --- | --- | --- |
| `bluetooth` | Bluetooth | you use [Room Assistant](/Pi-Guide/Room-Assistant.md), [Home Assistant](/Pi-Guide/Home-Assistant.md) Bluetooth integrations, or any Bluetooth device |
| `ModemManager` | Cellular/USB modems | you have a 4G/LTE modem plugged in |
| `triggerhappy` | Hotkeys from a connected keyboard | you use a keyboard's special keys on the Pi |

Disable the ones you don't need, for example:

```bash
sudo systemctl disable --now bluetooth ModemManager
sudo systemctl disable --now triggerhappy.service triggerhappy.socket
```

Then confirm nothing broke:

```bash
systemctl --failed
```

Undo: `sudo systemctl enable --now <service>`.

If you installed Samba for [file sharing](/Pi-Guide/NAS.md), it may also have added `samba-ad-dc`, a Windows domain controller you almost certainly don't want. It's normally not running. `sudo systemctl mask samba-ad-dc` makes sure it never starts. Leave `smbd` and `nmbd` alone: those *are* the file sharing.

## Turn Off WiFi on a Wired Pi

If the Pi is plugged into ethernet, WiFi is just a second way in, and a second IP address that other devices might end up using.

<ins>IMPORTANT:</ins> check which connection your SSH session is using first. If it's WiFi, turning WiFi off cuts you off immediately:

```bash
echo $SSH_CONNECTION                               # the first address is your computer
ip route get $(echo $SSH_CONNECTION | cut -d' ' -f1)   # "dev eth0" = wired, "dev wlan0" = WiFi
```

If it says `wlan0`, reconnect using the Pi's **wired** IP address (`ip -4 addr show eth0`) and check again.

- **Your DHCP reservation is per network card.** Ethernet and WiFi have different MAC addresses, so if you reserved the Pi's IP (see [Give the Pi a Static IP](/README.md#give-the-pi-a-static-ip)) while it was on WiFi, the reservation doesn't apply to ethernet. Reserve the address for `eth0`'s MAC (`ip link show eth0`, the `link/ether` value), and update anything that points at the old address.

Then turn WiFi off. NetworkManager remembers this across reboots:

```bash
sudo nmcli radio wifi off
nmcli radio   # WIFI should say "disabled"
```

Undo: `sudo nmcli radio wifi on`.

The trade-off: with WiFi off, **there's no backup connection**. If the cable, switch port or ethernet itself fails, getting back in means plugging in a keyboard and monitor, or pulling the SD card. For a Pi on a shelf that's usually fine. For one that's hard to reach, you might prefer to leave WiFi on.

## Update the Bootloader

The Pi 4 and Pi 5 keep their bootloader in a separate EEPROM chip, not on the SD card, so OS updates don't upgrade it on their own. Newer versions fix boot bugs and improve USB and power handling.

```bash
sudo rpi-eeprom-update        # compare CURRENT with LATEST
sudo rpi-eeprom-update -a     # stage the update if one is available
sudo reboot                   # it's flashed during this boot
```

Two things to know:

- Until you reboot, `rpi-eeprom-update` keeps saying **"UPDATE AVAILABLE"**, because `CURRENT` is the bootloader that's *running*. Don't run `-a` again because of it. After the reboot it should report the bootloader is up to date.
- A staged update is also reported by the [login status message](/Pi-Guide/Job-Monitoring.md#login-status-message), so it doesn't sit there forgotten.

## Check Temperature and Power

```bash
vcgencmd measure_temp
vcgencmd get_throttled
```

- The temperature should stay under about 80 °C under load. Above that the Pi slows itself down. A heatsink or the official active cooler fixes this.
- `throttled=0x0` means everything is fine. Anything else means the Pi has seen low voltage or overheating since it booted, and low voltage is a common cause of SD card corruption and random USB drive disconnects. Use the official power supply, and power USB drives from a powered hub (see [NAS](/Pi-Guide/NAS.md#configuration)).

## Sources

- https://www.raspberrypi.com/documentation/computers/raspberry-pi.html#raspberry-pi-bootloader-configuration
- https://www.raspberrypi.com/documentation/computers/os.html#vcgencmd
- https://docs.kernel.org/admin-guide/sysctl/vm.html#swappiness
- https://networkmanager.dev/docs/api/latest/nmcli.html
