# Unattended-Upgrades

Automatically update and upgrade Raspberry Pi.

## Table of Contents

- [Installation](#installation)
- [Sources](#sources)

## Installation

1. Enter:
   ```bash
   sudo apt-get update
   sudo apt-get install unattended-upgrades
   ```
1. Test with a dry run:
   ```bash
   sudo unattended-upgrade -d -v --dry-run
   ```
1. Enable by selecting `Yes`:
   ```bash
   sudo dpkg-reconfigure --priority=low unattended-upgrades
   ```

## Sources

- https://www.seancarney.ca/2021/02/06/secure-your-raspberry-pi-by-enabling-automatic-software-updates/
