# Watchdog

Reboot the Pi when there is a hardware failure. The Raspberry Pi has a hardware watchdog built in that will power cycle it if the chip is not refreshed within a certain interval.

## Table of Contents

- [Configuration](#configuration)
- [Testing](#testing)
- [Sources](#sources)

## Configuration

1. Check if you have `/dev/watchdog` by entering:
   ```
   ls -al /dev/watchdog*
   ```
   You should see something similar below:
   ```
   pi@pi4:~ $ ls -al /dev/watchdog*
   crw------- 1 root root  10, 130 Feb  9 23:22 /dev/watchdog
   crw------- 1 root root 250,   0 Feb  9 23:22 /dev/watchdog0
   ```
2. Enter:
   ```
   sudo nano /etc/systemd/system.conf
   ```
   Then uncomment by removing the `#` and set the following lines to:
   ```
   RuntimeWatchdogSec=15s
   RebootWatchdogSec=10min
   ```
   What the lines above mean:
   - `RuntimeWatchdogSec`: refresh the hardware watchdog every 15 seconds. If for some reason the refresh fails after 3 intervals, power cycle the system.
   - `RebootWatchdogSec`: on reboot, if the system takes more than 10 minutes, power cycle the system.
3. Reboot:
   ```
   sudo reboot
   ```

## Testing

Run a "fork bomb" on your shell:

```
:(){ :|:& };:
```

Running this code will render your Raspberry Pi inaccessible until it’s reset by the watchdog. The Pi should be back up and running after a few minutes.

## Sources

- https://raspberrypi.stackexchange.com/questions/99584/rpi-freezes-every-now-and-then-how-to-fix-it-with-a-watchdog
- https://diode.io/raspberry%20pi/running-forever-with-the-raspberry-pi-hardware-watchdog-20202/
