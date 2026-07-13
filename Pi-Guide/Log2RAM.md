# Log2Ram

Write system logs to RAM instead of the micro SD card, greatly reducing SD card wear (the most common cause of Pi failures). Logs are flushed to disk daily and on shutdown/reboot.

## Table of Contents

- [Installation](#installation)
- [Configuration](#configuration)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Installation

1. Add the azlux APT repository (the `$VERSION_CODENAME` variable picks the right release for your OS version):
   ```bash
   . /etc/os-release
   sudo wget -O /usr/share/keyrings/azlux-archive-keyring.gpg https://azlux.fr/repo.gpg
   echo "deb [signed-by=/usr/share/keyrings/azlux-archive-keyring.gpg] http://packages.azlux.fr/debian/ $VERSION_CODENAME main" | sudo tee /etc/apt/sources.list.d/azlux.list
   ```
1. Install Log2Ram:
   ```bash
   sudo apt update
   sudo apt install log2ram
   ```
1. Reboot to activate it:
   ```bash
   sudo reboot
   ```

## Configuration

The defaults are fine for most setups. To adjust, open the config:

```bash
sudo nano /etc/log2ram.conf
```

The main option is `SIZE` (default `128M`) — the amount of RAM reserved for `/var/log`. Raise it if you run chatty services like Pi-hole with heavy logging. Restart to apply:

```bash
sudo systemctl restart log2ram
```

## Testing

Check that `/var/log` is mounted in RAM:

```bash
df -hT | grep log2ram
systemctl status log2ram
```

You should see a `log2ram` tmpfs mounted on `/var/log`.

## Troubleshooting

If the service fails to start, your existing logs are likely bigger than `SIZE`. Either raise `SIZE` in the config, or clear old logs first:

```bash
sudo journalctl --vacuum-size=64M
sudo rm /var/log/*.gz /var/log/*.1 2>/dev/null
sudo systemctl restart log2ram
```

## Sources

- https://github.com/azlux/log2ram
