# Watchdog 
Reboot the Pi when there is a hardware failure. The Raspberry Pi has a hardware watchdog built in that will power cycle it if the chip is not refreshed within a certain interval.

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
    Then uncomment and set the following lines to:
    ```
    RuntimeWatchdogSec=10
    ShutdownWatchdogSec=10min
    ```
What the lines above say is:
* Refresh the hardware watchdog every 10 seconds. If for some reason the refresh fails (I believe after 3 intervals; i.e. 30s) power cycle the system.
* On shutdown, if the system takes more than 10 minutes to reboot, power cycle the system.
3. Reboot:
    ```
    sudo reboot
    ``` 

## Testing
_Optional:_ run a "fork bomb" on your shell:
```
:(){ :|:& };:
```
Running this code will render your Raspberry Pi inaccessible until it’s reset by the watchdog. The Pi should be back up and running after a few minutes. If you notice your Pi is a little slow try rebooting it again with `sudo reboot`.

## Sources
* https://raspberrypi.stackexchange.com/questions/99584/rpi-freezes-every-now-and-then-how-to-fix-it-with-a-watchdog
* https://diode.io/raspberry%20pi/running-forever-with-the-raspberry-pi-hardware-watchdog-20202/