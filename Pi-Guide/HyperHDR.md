# HyperHDR

Fork of Hyperion, with improvements, notably to HDR colour accuracy and brightness. Used to sync smart lights to picture on display. Far cheaper alternative to the Hue Sync Box (~$300).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Testing](#testing)
- [Updating](#updating)
- [Sources](#sources)

## Prerequisites

If you have Hyperion installed, uninstall it:

```
sudo apt-get --purge autoremove hyperion
```

## Installation

```
type -p curl >/dev/null || sudo apt install curl -y
curl -fsSL https://awawa-dev.github.io/hyperhdr.public.apt.gpg.key | sudo dd of=/usr/share/keyrings/hyperhdr.public.apt.gpg.key \
&& sudo chmod go+r /usr/share/keyrings/hyperhdr.public.apt.gpg.key \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hyperhdr.public.apt.gpg.key] https://awawa-dev.github.io $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hyperhdr.list > /dev/null \
&& sudo apt update \
&& sudo apt install hyperhdr -y
```

## Testing

You can open the HyperHDR website by heading to `[PIIPADDRESS]:8090`

## Updating

1. First remove the HyperHDR install:
   ```
   sudo apt remove hyperhdr
   ```
2. Repeat the [Installation](#installation) steps again. Your settings will be saved.

## Sources

- https://github.com/awawa-dev/HyperHDR/wiki/Installation#Linux
