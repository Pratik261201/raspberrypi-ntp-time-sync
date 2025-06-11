# Microsecond Accurate NTP with Raspberry Pi and PPS GPS

## Introduction

Lots of acronyms in that title. If I expand them out, it says – “microsecond accurate network time protocol with a Raspberry Pi and pulse per second global positioning system \[receiver]”. What it means is you can get super accurate timekeeping (1 microsecond = 0.000001 seconds) with a Raspberry Pi and a GPS receiver (the GT-U7 is less than \$12) that spits out pulses every second. By following this guide, you will have your very own Stratum 1 NTP server at home!

## Why We Need Time This Accurate

You don’t. There aren’t many applications for this level of timekeeping in general, and even fewer at home. But this blog is called Austin’s Nerdy Things so here we are. Using standard, default internet NTP these days will get your computers to within a 10–20 milliseconds of actual time (1 millisecond = 0.001 seconds). By default, Windows computers get time from `time.windows.com`. macOS computers get time from `time.apple.com`. Linux devices get time from `<entity>.pool.ntp.org`, like `debian.pool.ntp.org`. PPS gets you to the next SI prefix in terms of accuracy (milli → micro), which means 1000× more accurate timekeeping.

## Materials Needed

* Raspberry Pi 5
* GPS with PPS output – Adafruit Ultimate GPS (\$40ish)
* A GPS antenna (for getting better GPS satellite reception)
* Wires to connect it all up (5 wires needed – 5V/RX/TX/GND/PPS)

## Folder Structure

```bash
mkdir Images files
```

* `Images/` – For wiring and result screenshots
* `files/` – For configuration backups and logs

## 0 – Update Your Pi and Install Packages

You should update your Pi to latest before basically any project. We will install some other packages as well. `pps-tools` help us check that the Pi is receiving PPS signals from the GPS module. We also need `gpsd` for the GPS decoding of both time and position. I use `chrony` instead of `ntpd` because it seems to sync faster than NTPd in most instances and also handles PPS without compiling from source (the default Raspbian NTP doesn’t do PPS). Installing `chrony` will remove `ntpd`.

```bash
sudo apt update
sudo apt upgrade
sudo rpi-update
sudo apt install pps-tools gpsd gpsd-clients python-gps chrony
```

## 1 – Add GPIO and Module Info Where Needed

In `/boot/config.txt`, add `dtoverlay=pps-gpio,gpiopin=18` to a new line. This is necessary for PPS. If you want to get the NMEA data from the serial line, you must also enable UART and set the initial baud rate.

```bash
sudo bash -c "echo '# the next 3 lines are for GPS PPS signals' >> /boot/firmware/config.txt"
sudo bash -c "echo 'dtoverlay=pps-gpio,gpiopin=18' >> /boot/firmware/config.txt"
sudo bash -c "echo 'enable_uart=1' >> /boot/firmware/config.txt"
sudo bash -c "echo 'init_uart_baud=9600' >> /boot/firmware/config.txt"
```

In `/etc/modules`, add `pps-gpio` to a new line.

```bash
sudo bash -c "echo 'pps-gpio' >> /etc/modules"
```

Reboot:

```bash
sudo reboot
```

## 2 – Wire up the GPS Module to the Pi

I used the Adafruit Ultimate GPS breakout. It has 9 pins but we are only interested in 5. There is also the Adafruit GPS hat which fits right on the Pi but that seems expensive for what it does (but it is significantly neater in terms of wiring).

**Pin connections:**

* GPS PPS to RPi pin 12 (GPIO 18)
* GPS VIN to RPi pin 2 or 4
* GPS GND to RPi pin 6
* GPS RX to RPi pin 8
* GPS TX to RPi pin 10

![Wiring Diagram][Images/step2-connection-visual]

## 2.5 – Free up the UART Serial Port for the GPS Device

Run `raspi-config` → **3 – Interface Options** → **I6 – Serial Port** → **Would you like a login shell to be available over serial?** → No → **Would you like the serial port hardware to be enabled?** → Yes.
![][Images/uart]
Finish and reboot.

## 3 – Check That PPS Is Working

First, check that the PPS module is loaded:

```bash
lsmod | grep pps
```

Expected output:

```
pps_gpio               16384  0
pps_core               16384  1 pps_gpio
```

Second, check for the PPS pulses:

```bash
sudo ppstest /dev/pps0
```

Expected output (a new line every second):

```
trying PPS source "/dev/pps0"
found PPS source "/dev/pps0"
ok, found 1 source(s), now start fetching data...
source 0 - assert 1618799061.999999504, sequence: 882184 - clear  0.000000000, sequence: 0
source 0 - assert 1618799062.999999305, sequence: 882185 - clear  0.000000000, sequence: 0
source 0 - assert 1618799063.999997231, sequence: 882186 - clear  0.000000000, sequence: 0
^C
```

## 4 – Change GPSd to Start Immediately Upon Boot

Edit `/etc/default/gpsd` and change:

```diff
- GPSD_OPTIONS=""
+ GPSD_OPTIONS="-n"
- DEVICES=""
+ DEVICES="/dev/ttyS0 /dev/pps0"
```

My full `/etc/default/gpsd`:

```bash
pi@raspberrypi:~ $ sudo cat /etc/default/gpsd
# Default settings for the gpsd init script and the hotplug wrapper.
# Start the gpsd daemon automatically at boot time
START_DAEMON="true"
# Use USB hotplugging to add new USB devices automatically to the daemon
USBAUTO="true"
# Devices gpsd should collect to at boot time.
# They need to be read/writeable, either by user gpsd or the group dialout.
DEVICES="/dev/tserial0 /dev/pps0"
# Other options you want to pass to gpsd
GPSD_OPTIONS="-n"
```

```bash
sudo reboot
```

## 4.5 – Check GPS for Good Measure

To ensure your GPS has a valid position, you can run `gpsmon` or `cgps` to check satellites and such. This check also ensures GPSd is functioning as expected. If your GPS doesn’t have a position solution, you won’t get a good time signal. If GPSd isn’t working, you won’t get any updates on the screen. The top portion will show the analyzed GPS data and the bottom portion will scroll by with the raw GPS sentences from the GPS module.

![GPS Fix Result][Images/result1]

## 4.6– Edit Chrony Config Files

**2025 update:** see the newer post on Chrony config for how to get single source (i.e. just your single GPS to use both NMEA and PPS) timing to work successfully.

My entire `/etc/chrony/chrony.conf`:

```chrony
refclock SHM 0 offset 0.325 delay 0.2 refid GPS
refclock PPS /dev/pps0 lock GPS refid PPS prefer

server 10.98.1.1 iburst minpoll 3 maxpoll 5
server time-a-b.nist.gov iburst
server time-d-b.nist.gov
server utcnist.colorado.edu
server time.windows.com
server time.apple.com

allow 192.168.129.0/24
keyfile /etc/chrony/chrony.keys
driftfile /var/lib/chrony/chrony.drift
log tracking measurements statistics
logdir /var/log/chrony
maxupdateskew 100.0
hwclockfile /etc/adjtime
rtcsync
makestep 1 3
```

Restart Chrony and wait a few minutes:

```bash
sudo systemctl restart chrony
```


# Verifying NTP Server Using GPS PPS

This document covers the verification of your Raspberry Pi Stratum‑1 NTP server using GPS PPS, including commands and log interpretation.

## 5 – Verify the NTP Server Is Using the GPS PPS

5.1. **Restart Chrony**

   ```bash
   sudo systemctl restart chrony
   ```

5.2. **Check sources immediately**

   Run:

   ```bash
   chronyc sources
   ```

   Expected output:

   ```bash
   210 Number of sources = 9
   MS Name/IP address         Stratum Poll Reach LastRx Last sample
   ===============================================================================
   #? NMEA                          0   4     0     -     +0ns[   +0ns] +/-    0ns
   #? PPS                           0   4     0     -     +0ns[   +0ns] +/-    0ns
   ^? pfsense.home.fluffnet.net     0   3     3     -     +0ns[   +0ns] +/-    0ns
   ^? time-a-b.nist.gov             1   6     3     1  -2615us[-2615us] +/- 8218us
   ^? time-d-b.nist.gov             1   6     1     3  -2495us[-2495us] +/- 7943us
   ^? india.colorado.edu            0   6     0     -     +0ns[   +0ns] +/-    0ns
   ^? 13.86.101.172                 3   6     1     4  -4866us[-4866us] +/-   43ms
   ^? usdal4-ntp-002.aaplimg.c>     1   6     1     4  -2143us[-2143us] +/-   13ms
   ^? time.cloudflare.com           3   6     1     3  -3747us[-3747us] +/- 9088us
   ```

   * `#` means a locally connected source of time.
   * `?` indicates Chrony is still determining the status of each source.

5.3. **Check sources after a few minutes**

   Run:

   ```bash
   chronyc -n sources
   ```

   Expected output:

   ```bash
   210 Number of sources = 9
   MS Name/IP address         Stratum Poll Reach LastRx Last sample
   ===============================================================================
   #x NMEA                          0   4   377    23    -37ms[  -37ms] +/- 1638us
   #* PPS                           0   4   377    25   -175ns[ -289ns] +/-  126ns
   ^? 10.98.1.1                     0   5   377     -     +0ns[   +0ns] +/-    0ns
   ^- 132.163.96.1                  1   6   177    22  -3046us[-3046us] +/- 8233us
   ^- 2610:20:6f96:96::4            1   6    17    28  -2524us[-2524us] +/- 7677us
   ^? 128.138.140.44                1   6     3    30  -3107us[-3107us] +/- 8460us
   ^- 13.86.101.172                 3   6    17    28  -8233us[-8233us] +/-   47ms
   ^- 17.253.2.253                  1   6    17    29  -3048us[-3048us] +/-   14ms
   ^- 2606:4700:f1::123             3   6    17    29  -3325us[-3325us] +/- 8488us
   ```

   * In the **S** column:

     * `*` means the active source.

     * `+` means a good candidate if the current one fails.

     * `x` means a false ticker (not used).

   > **Observation:** The `PPS` line shows `*`, indicating PPS is the active, preferred source with sub-µs accuracy.

5.4. **Ping a remote source**

   To gauge network delay, run:

   ```bash
   ping -c 5 17.253.2.253
   ```

   Sample output:

   ```bash
   PING 17.253.2.253 (17.253.2.253) 56(84) bytes of data.
   64 bytes from 17.253.2.253: icmp_seq=1 ttl=54 time=25.2 ms
   64 bytes from 17.253.2.253: icmp_seq=2 ttl=54 time=27.7 ms
   64 bytes from 17.253.2.253: icmp_seq=3 ttl=54 time=23.8 ms
   64 bytes from 17.253.2.253: icmp_seq=4 ttl=54 time=24.4 ms
   64 bytes from 17.253.2.253: icmp_seq=5 ttl=54 time=23.4 ms
   --- 17.253.2.253 ping statistics ---
   5 packets transmitted, 5 received, 0% packet loss, time 4007ms
   rtt min/avg/max/mdev = 23.403/24.954/27.780/1.547 ms
   ```

   Note how the average RTT (\~25 ms) compares to the PPS offset (< 1 µs).

5.5. **View the full `tracking.log`**

   To examine the complete clock discipline log, run:

   ```bash
   sudo cat /var/log/chrony/tracking.log
   ```

   This will dump all timestamped adjustments and help you analyze long-term stability.

---

**Congratulations!** You’ve confirmed that your Raspberry Pi NTP server is locked to GPS PPS and achieving micro‑ to nanosecond‑level accuracy. 🎉


## 6 – Results and Interpretation

Here’s how to read and interpret your `tracking.log` output, with the key fields and what they tell you about your chrony-disciplined clock:
![Results][Images/result5]
```
===================================================================================================================================
   Date (UTC) Time     IP Address   St   Freq ppm   Skew ppm     Offset L Co  Offset sd Rem. corr. Root delay Root disp. Max. error
===================================================================================================================================
```

| Column          | Meaning                                                                       |
| --------------- | ----------------------------------------------------------------------------- |
| Date (UTC) Time | When chrony sampled/adjusted the clock (always in UTC).                       |
| IP Address      | Which source was used:                                                        |
|                 | • `0.0.0.0` = no source yet (initial boot)                                    |
|                 | • `GPS` = NMEA-derived time from gpsd (Stratum 1 shared memory)               |
|                 | • `PPS` = the 1-pulse-per-second hardware signal (your most precise source)   |
|                 | • other IPs = remote NTP servers                                              |
| St              | “Stratum” of the source (0 or 1 for GPS/PPS, >1 for internet servers)         |
| Freq ppm        | The current clock frequency adjustment, in parts-per-million (how fast/slow). |
| Skew ppm        | The estimated uncertainty (jitter) in that frequency setting.                 |
| Offset          | The instantaneous difference between your system clock and the source:        |
|                 | • Reported in scientific notation (e.g. `6.871e-01` = 0.6871 s = 687.1 ms)    |
|                 | • Later values like `-3.253e-05` = –32.53 µs, or `-6.041e-06` = –6.041 µs     |
| L               | “Lock” indicator: `N` = locked to this source.                                |
| Co              | Count of consecutive samples used from this source.                           |
| Offset sd       | Standard deviation of recent offsets (stability measure).                     |
| Rem. corr.      | Total cumulative correction applied so far (seconds).                         |
| Root delay      | Round-trip network or source delay (seconds).                                 |
| Root disp.      | Estimate of total dispersion (uncertainty) from this source.                  |
| Max. error      | The maximum possible error (seconds) chrony allows for this source.           |

### Walk-through of Your Log

**Initial Bootstrap**

```sql
2025-06-09 14:06:33  0.0.0.0 … Offset 0.000e+00 ? …  
```

No source yet; clock not synced.

**First GPS NMEA Sample**

```mathematica
2025-06-09 14:07:36  GPS … Offset 6.871e-01 N …  
```

The GPS NMEA sentence is \~0.687 s ahead of the (unsynced) system clock. Chrony sees it but will not lock solely on GPS NMEA.

**First PPS Lock**

```mathematica
2025-06-09 14:07:35  PPS … Offset 3.111e-01 N …  
```

The first PPS edge shows the system clock \~0.311 s off PPS; chrony begins to discipline toward PPS.

**Rapid Convergence**
By 14:07:52, offset is -3.253e-05 s = –32.53 µs.
By 14:08:08, offset is -2.146e-05 s = –21.46 µs.
By 14:08:24, offset is -1.301e-05 s = –13.01 µs.
…and so on, dropping into single-digit microseconds within a minute.

**Nanosecond-Level Precision**
After about 14:09:12, the offset column shows values like `1.587e-06 s` (1.587 µs), and then quickly to sub-microsecond:

```
e.g. 14:09:28 → 5.396e-06 s = 5.396 µs
```

But soon you’ll see lines where offset is `-6.041e-06 s` (–6.041 µs), and then further settling toward hundreds of nanoseconds:

A few entries later you’ll see offsets reported like `-2.146e-06` (–2.146 µs) and ultimately ±1e-09 in the Root delay and extremely low Root dispersion.

**Stable Long-Term Sync**
Through the next pages of your log, PPS remains the preferred source (L = N, Co = 1) and offset hovers in the ±10 µs to ±10 ns range. That’s atomic-clock level discipline!

### Summary

* **Lock time:** \~1 minute to converge from hundreds of milliseconds to micro/nanoseconds.
* **Best accuracy:** Your system clock now stays within a few hundred nanoseconds of the GPS PPS pulse.
* **Jitter:** The small `Offset sd` (\~1e-09) shows extremely low variation.
* **Network servers remain idle** (?) or show `-` because PPS is far more precise.

Refrences :
* https://austinsnerdythings.com/2021/04/19/microsecond-accurate-ntp-with-a-raspberry-pi-and-pps-gps/
* https://www.satsignal.eu/ntp/Raspberry-Pi-NTP.html
* https://www.chrony.eu/setup/linux#servers

🎉 You’ve successfully created a nanosecond-accurate, Stratum-1 NTP server on Raspberry Pi! 🎉

---

<!-- Image references -->

[Images/step2-connection-visual]: ./Images/step2-connection-visual.jpg
[Images/uart]: ./Images/uart.png
[Images/result1]: ./Images/result1.png
[Images/result5]: ./Images/result5.png
