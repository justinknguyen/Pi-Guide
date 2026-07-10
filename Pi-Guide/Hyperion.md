# Hyperion

Used to sync smart lights to picture on display. Far cheaper alternative to the Hue Sync Box (~$300).

## Table of Contents

- [Prerequisites](#prerequisites)
- [Installation](#installation)
- [Configuration](#configuration)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## Prerequisites

- [USB 2.0 1080p 30Hz Capture Card](https://www.amazon.ca/gp/product/B08F6ZD2RK/ref=ppx_yo_dt_b_asin_title_o00_s00?ie=UTF8&psc=1)
- [HDMI Splitter, 1-in-2-out, 4k 60Hz Simultaneous Output](https://a.co/d/fNABOwW)
- 3 HDMI Cables

Total cost: ~$50 <br><br>
One HDMI cable runs from the input device (e.g., PS5 or PC) to the splitter's input port, so it can feed both the monitor/tv and the Raspberry Pi at once (that's why the splitter needs two output ports). The second cable connects one splitter output to the monitor/tv. The third connects the other splitter output to the USB Capture Card, which then connects to the Raspberry Pi.

## Installation

1. Install the required dependencies:
   ```bash
   sudo apt-get update && sudo apt-get install wget gpg apt-transport-https lsb-release
   ```
1. Import the Hyperion public key and add it as an APT source:
   ```bash
   wget -qO- https://apt.hyperion-project.org/hyperion.pub.key | sudo gpg --dearmor -o /usr/share/keyrings/hyperion.pub.gpg
   echo "deb [signed-by=/usr/share/keyrings/hyperion.pub.gpg] https://apt.hyperion-project.org/ $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hyperion.list
   ```
1. Update the package list and install Hyperion:
   ```bash
   sudo apt-get update && sudo apt-get install hyperion
   ```

## Configuration

Once Hyperion is installed on the Pi, you can access the Hyperion website using `[PIIPADDRESS]:8090`.

1. Go to LED Instances > LED Output, and choose the controller type depending on your smart lights (e.g., philipshue), then proceed with the setup.
1. Go to Capturing Hardware and activate the USB Capture option.
1. To reduce capture delay, set device resolution to 640x480 and FPS to 30.
   <br><br>
   To turn it off when it detects rainbow bars, use the following settings under `Capturing Hardware`:

   | Setting | Value |
   | --- | --- |
   | Signal detection | checked |
   | Red signal threshold | 0 |
   | Green signal threshold | 100 |
   | Blue signal threshold | 0 |
   | Signal Counter Threshold | 200 |
   | Signal Detection VMin | 0 |
   | Signal Detection VMax | 1 |
   | Signal Detection HMin | 0.4 |
   | Signal Detection HMax | 0.45 |

## Troubleshooting

If you get the following error when trying to update your Pi:

```
W: Failed to fetch https://apt.hyperion-project.org/dists/bullseye/InRelease  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY 21710A7427490076
```

import the public key again and then you should be able to update and upgrade your Pi:

```bash
wget -qO- https://apt.hyperion-project.org/hyperion.pub.key | sudo gpg --dearmor -o /usr/share/keyrings/hyperion.pub.gpg
```

## Sources

- https://docs.hyperion-project.org/en/user/Installation.html#raspberry-pi
- https://www.youtube.com/watch?v=JvcR2td1Cso
- https://hyperion-project.org/forum/index.php?thread/11231-new-1080p-grabber-possibly-causing-delay/
- https://hyperion-project.org/forum/index.php?thread/12170-how-to-turn-off-the-rainbow-signal-when-there-is-no-signal-on-the-hdmi-grabber/
