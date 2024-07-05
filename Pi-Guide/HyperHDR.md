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

1. cd to the tmp directory:
   ```
   cd /tmp
   ```
2. Download HyperHDR to the directory (use the latest version, current is v20.0.0.0)
   ```
   wget https://github.com/awawa-dev/HyperHDR/releases/download/v20.0.0.0/HyperHDR-20.0.0.0-Linux-aarch64.deb
   ```
   - if you are on the Raspberry Pi 32-bit OS, replace `aarch64` with `armv6l`
   - if you are using a generic Linux machine, replace `aarch64` with `x86_64`
3. Install HyperHDR:
   ```
   sudo apt install ./HyperHDR-20.0.0.0-Linux-aarch64.deb
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
