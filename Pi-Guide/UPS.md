# UPS

Shut down every machine on a UPS cleanly before its battery runs out, and have them all **power back on by themselves** when the power returns. This uses Network UPS Tools (NUT): the machine the UPS's USB cable plugs into (the **UPS host**) reads the battery and shares its status over the network, and every other machine on the same UPS (Pis, a NAS, another server) listens to it.

The UPS host can be a Pi or any Debian-based machine; the commands are the same. The examples use a CyberPower CP1500AVRLCD3 on a server at `192.168.50.197`, powering two Pis and a NAS.

## Table of Contents

- [How It Works](#how-it-works)
- [Set Up the UPS Host](#set-up-the-ups-host)
- [Power Back On After an Outage](#power-back-on-after-an-outage)
- [Set Up Each Pi as a Client](#set-up-each-pi-as-a-client)
- [Add a NAS](#add-a-nas)
- [Testing](#testing)
- [Troubleshooting](#troubleshooting)
- [Sources](#sources)

## How It Works

- The UPS host runs three parts: the **driver** (talks to the UPS over USB), **upsd** (serves its status on port 3493), and **upsmon** (decides when to shut down). Each client runs only upsmon.
- Nothing shuts down just because the power went out. upsmon waits until the UPS reports **on battery *and* low battery** (`OB LB`). The UPS decides "low" itself, usually at 10% charge or 5 minutes of runtime left, whichever comes first. At a light load that can be hours into an outage, so short outages are ridden out without anything turning off.
- At that point the host sends a **forced shutdown** (FSD) to every client, gives them a moment to start shutting down, then shuts itself down. As the very last step, it tells the UPS to **switch its outlets off after a delay, and back on when mains power returns**. That last step is what brings everything back up. Without it, a machine that shut down while the UPS still had battery stays off even after the power comes back (see [Power Back On After an Outage](#power-back-on-after-an-outage)).

## Set Up the UPS Host

1. Plug the UPS's USB cable into the host, then check it's detected:
   ```bash
   lsusb
   ```
   You should see a line similar to:
   ```
   Bus 001 Device 002: ID 0764:0601 Cyber Power System, Inc. PR1500LCDRT2U UPS
   ```
   The two numbers after `ID` are the vendor and product IDs.
1. Install NUT:
   ```bash
   sudo apt install nut
   ```
1. Set the mode, so upsd serves other machines:
   ```bash
   sudo nano /etc/nut/nut.conf
   ```
   ```ini
   MODE=netserver
   ```
1. Describe the UPS:
   ```bash
   sudo nano /etc/nut/ups.conf
   ```
   Add at the bottom (use your own IDs from `lsusb`):
   ```ini
   [UPS]
       driver = usbhid-ups
       port = auto
       vendorid = "0764"
       productid = "0601"
       desc = "UPS on the server"
       offdelay = 120
       ondelay = 180
   ```
   - `[UPS]` is the name clients use to refer to it (`UPS@<host-ip>`).
   - `usbhid-ups` covers most USB UPSes (CyberPower, APC, Eaton). If yours isn't detected, `sudo nut-scanner -U` prints a ready-made block to paste instead.
   - `offdelay` and `ondelay` are explained in [Power Back On After an Outage](#power-back-on-after-an-outage). Don't skip them.
1. Let upsd listen on the network:
   ```bash
   sudo nano /etc/nut/upsd.conf
   ```
   ```ini
   LISTEN 0.0.0.0 3493
   ```
1. Create the accounts. Generate a password for each with `openssl rand -hex 12`:
   ```bash
   sudo nano /etc/nut/upsd.users
   ```
   ```ini
   [monuser]
       password = <password>
       upsmon primary

   [upsclient]
       password = <another-password>
       upsmon secondary
   ```
   `monuser` is for the host itself. `upsclient` is for the Pis: a *secondary* can only listen, so a leaked client password can't be used to shut everything down. (NUT older than 2.8 calls these `master` and `slave`; 2.8 accepts both spellings.)
1. Tell the host's upsmon what to watch:
   ```bash
   sudo nano /etc/nut/upsmon.conf
   ```
   Add or change these lines:
   ```ini
   MONITOR UPS@localhost 1 monuser <password> primary
   SHUTDOWNCMD "/sbin/shutdown -h +0"
   POWERDOWNFLAG /etc/killpower
   ```
   `POWERDOWNFLAG` is required: it's the file the final shutdown step checks before telling the UPS to cycle its outlets.
1. These files hold passwords, so make them readable only by root and NUT:
   ```bash
   sudo chown root:nut /etc/nut/*.conf /etc/nut/upsd.users
   sudo chmod 640 /etc/nut/*.conf /etc/nut/upsd.users
   ```
1. If you use ufw (see [SSH Hardening](/Pi-Guide/SSH-Hardening.md)), allow port 3493 from your LAN only:
   ```bash
   sudo ufw allow from 192.168.50.0/24 to any port 3493 proto tcp comment 'NUT upsd'
   ```
   NUT's protocol is unencrypted, so never expose this port to the internet.
1. Start everything:
   ```bash
   sudo systemctl restart nut-driver-enumerator nut-server nut-monitor
   ```
1. Check the UPS is being read:
   ```bash
   upsc UPS
   ```
   Look for these lines:
   ```
   battery.charge: 100
   battery.runtime: 11100
   ups.delay.shutdown: 120
   ups.delay.start: 180
   ups.status: OL
   ```
   `OL` means on line (mains) power, `OB` means on battery, `LB` means low battery. `battery.runtime` is in seconds, so 11100 is just over 3 hours at the current load.

## Power Back On After an Outage

Getting everything to shut down is only half of it. Picture the outage ending *after* your machines shut down but *before* the battery ran flat. The UPS never lost output, so from each machine's point of view the power never went away, and a machine that's been shut down stays off until someone presses its button. That's why the host tells the UPS to cycle its outlets as its very last step. Three things have to be right for that to work.

**1. The UPS host must actually send the command.** Debian's NUT installs a final shutdown hook (`/lib/systemd/system-shutdown/nutshutdown`) that sends it whenever `/etc/killpower` exists, which upsmon creates during a power-failure shutdown. Two settings silently stop it:

- `sdorder = -1` in `ups.conf` excludes that UPS from the command entirely. If a guide or `nut-scanner` added it, remove it.
- A missing `POWERDOWNFLAG` in `upsmon.conf`.

Check with a dry run, which changes nothing:

```bash
sudo upsdrvctl -t shutdown
```

It should include a line ending in `-a UPS -k`. If that line is missing, the UPS will never be told to cycle.

**2. The delays must fit your machines.**

- `offdelay`: how long after the command the UPS cuts its outlets. The default is 20 seconds, and it starts counting when the **host** is almost done shutting down, not when the clients are. A Pi running Docker, or a NAS, can easily need longer than that, so use **120**.
- `ondelay`: how long after mains returns the UPS waits before switching its outlets back on. It must be longer than `offdelay`; **180** works.
- CyberPower units round these to whole minutes, so use multiples of 60.
- It all has to fit in the battery left at "low battery" (5 minutes on most units). At 120 seconds there's still margin.

After changing them, restart the driver and confirm the UPS took the values:

```bash
sudo systemctl restart nut-driver@UPS
upsc UPS | grep delay
```

**3. Each machine must boot when it gets power.**

- **Raspberry Pi:** always boots when power is applied. Nothing to do.
- **Mini PC / server:** set it in the BIOS, since Linux can't change it. Enter the BIOS (usually **Del** or **F2** at the logo, or `sudo systemctl reboot --firmware-setup` to boot straight into it) and look for **State After G3** (set to **S0 State**) or **Restore on AC Power Loss** (set to **Power On**). On Beelink mini PCs it's under **Chipset → PCH-IO Configuration**.
- **NAS:** look for an "auto power on after power failure" option in its power settings.

## Set Up Each Pi as a Client

1. Install only the client:
   ```bash
   sudo apt install nut-client
   ```
1. Set the mode:
   ```bash
   sudo nano /etc/nut/nut.conf
   ```
   ```ini
   MODE=netclient
   ```
1. Point upsmon at the UPS host:
   ```bash
   sudo nano /etc/nut/upsmon.conf
   ```
   ```ini
   MONITOR UPS@192.168.50.197 1 upsclient <another-password> secondary
   SHUTDOWNCMD "/sbin/shutdown -h +0"
   ```
   On Raspberry Pi OS Bullseye or older (NUT 2.7), write `slave` instead of `secondary`. Check your version with `upsmon -V`.
1. Lock the files down and restart:
   ```bash
   sudo chown root:nut /etc/nut/nut.conf /etc/nut/upsmon.conf
   sudo chmod 640 /etc/nut/nut.conf /etc/nut/upsmon.conf
   sudo systemctl restart nut-monitor
   ```
1. Check it can read the UPS:
   ```bash
   upsc UPS@192.168.50.197 ups.status
   ```
   It should print `OL`.

The Pi now shuts down when the host tells it to, or on its own if it sees `OB LB`.

## Add a NAS

Most NAS systems have a built-in NUT client, usually under a UPS or Power page, as "network UPS", "NUT server" or "UPS slave".

- Enter the UPS host's IP. If there's a UPS name field, use `UPS`. If there are username and password fields, use `upsclient`.
- If it offers "shut down after X seconds on battery" or **"until low battery"**, choose low battery so it matches everything else.
- Turn on the NAS's auto power-on after power failure as well.

**Many NAS systems only ask for an IP address** and log in with a fixed username. Synology DSM uses `monuser` with the password `secret`. TerraMaster TOS ("TNAS UPS server") was also seen logging in as `monuser`. Such a NAS can only use the `monuser` account, so `monuser`'s password in `upsd.users` must match what the NAS sends, and the host's `MONITOR` line must use the same password. If your NAS doesn't show up as connected, see [the NAS won't connect](#the-nas-wont-connect) below to find out exactly which credentials it sends.

Once it's connected, the UPS host lists it:

```bash
upsc -c UPS
```

```
192.168.50.87
192.168.50.103
192.168.50.89
127.0.0.1
```

## Testing

**Safe checks** (these change nothing):

```bash
upsc UPS ups.status          # OL = on mains
upsc -c UPS                  # every client should be listed
sudo upsdrvctl -t shutdown   # must show "-a UPS -k"
journalctl -u nut-monitor -n 20
```

**The real test** runs the whole sequence immediately, as if the battery had gone low:

```bash
sudo upsmon -c fsd
```

> **Warning:** this really shuts down every machine on the UPS, and the UPS then cuts all its outlets for a few minutes. Save open work and do it when downtime is fine.

What should happen:

1. Every client shuts down, then the host does.
1. About 2 minutes later (`offdelay`), the UPS switches its outlets off.
1. Since mains is present, about 3 minutes after that (`ondelay`), it switches them back on and everything boots by itself.

If everything stays off, see [Nothing came back on after the test or an outage](#nothing-came-back-on-after-the-test-or-an-outage) below.

## Troubleshooting

### A client says `connect failed`

- Check upsd is listening on the network, not just localhost: `sudo ss -tlnp | grep 3493` on the host should show `0.0.0.0:3493`.
- Check the firewall rule allows the client's subnet.
- If the UPS host is down, clients just keep retrying every few seconds. Until it's back, **they get no power warnings at all**, so a client that's running during an outage while the host is off won't shut down cleanly.

### `upsc` shows `Data stale` or `Driver not connected`

The driver lost the USB connection. Unplug and replug the UPS's USB cable, then run `sudo systemctl restart nut-driver@UPS`.

### The log keeps saying `OL+DISCHRG`

```
ups_status_set: seems that UPS [UPS] is in OL+DISCHRG state now. Is it calibrating or do you perhaps want to set 'onlinedischarge' option?
```

This is a known CyberPower quirk. It's harmless as long as `upsc UPS ups.status` normally shows `OL`.

### `Init SSL without certificate database`

This is printed by `upsc` and is harmless: NUT is simply running without TLS.

### Nothing came back on after the test or an outage

- Run `sudo upsdrvctl -t shutdown` and check it shows `-a UPS -k`. If not, look for `sdorder = -1` in `ups.conf`, or a missing `POWERDOWNFLAG`.
- Check `upsc UPS | grep delay` shows your `offdelay`/`ondelay`, not 20/30.
- Check each machine's BIOS or NAS power-on setting.
- Some UPS firmware won't switch its outlets back on if mains was present the whole time, which is exactly the situation in a `upsmon -c fsd` test. Try a larger `ondelay`. A real outage (unplugging the UPS from the wall and waiting for low battery) is the final word.

### The NAS won't connect

If your NAS has no username field and doesn't appear in `upsc -c UPS`, watch what it sends when you click Apply. NUT's protocol is plain text, so this shows the username and password:

```bash
sudo tcpdump -i any -A -s 0 'tcp port 3493 and host 192.168.50.87' | grep -aE 'USERNAME|PASSWORD|LOGIN|ERR'
```

Then add an account with those credentials to `/etc/nut/upsd.users` and run `sudo systemctl reload nut-server`. Stop the capture afterwards, and don't save its output: it contains the password.

## Sources

- https://networkupstools.org/docs/user-manual.chunked/index.html
- https://networkupstools.org/docs/man/ups.conf.html
- https://networkupstools.org/docs/man/usbhid-ups.html
- https://networkupstools.org/docs/man/upsd.users.html
- https://networkupstools.org/docs/man/upsmon.conf.html
- https://networkupstools.org/docs/man/upsdrvctl.html
