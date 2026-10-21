---
title: "TIL: Getting a first-gen Raspberry Pi online is the hard part (Garage Door Project, Part 2/5)"
date: 2026-07-07
description: 'My Pi is a first-generation Model B: no Wi-Fi, two USB ports and a very old Raspbian. Getting it onto my network turned into the longest chapter of the project.'
series: "Opening my garage door with my phone"
tags: [raspberry-pi, wifi, networking, ssh, til]
---

**Series:** Opening my garage door with my phone — Part 2 of 5
**Learned:** 4–7 July 2026

In Part 1 I decided a Raspberry Pi would host the API and talk to the Arduino over USB. The Pi I had is a **first-generation Model B**: no Wi-Fi, two USB ports, and a very old Raspbian on the SD card. Getting it reachable on my network turned into the longest chapter of the whole project.

## 1. Attempt one: a USB Wi-Fi dongle

**Buying advice I got (and would repeat):**

- A Pi 1 needs a **USB Wi-Fi adapter**. Look for "Linux compatible" or "Raspberry Pi compatible" and a chipset with in-kernel drivers.
- Prefer a small **2.4 GHz "nano"** adapter. The Pi 1 can't use 5 GHz speeds anyway and has very little power to spare on its USB ports.
- Use a **good power supply** and, ideally, a **powered USB hub**.

**What actually happened:**

1. `wpa_cli -i wlan0 reconfigure` failed with `Failed to connect to wpa_supplicant - wpa_ctrl_open: No such file or directory`.
2. `sudo raspi-config` had **no network options at all**. That's the tell that you're on a very old Raspbian (Wheezy/Jessie era). Back then Wi-Fi was configured in `/etc/network/interfaces`:

   ```
   auto wlan0
   allow-hotplug wlan0
   iface wlan0 inet dhcp
       wpa-ssid "YOUR_SSID"
       wpa-psk "YOUR_PASSWORD"
   ```

3. Still nothing, so I checked whether the hardware was even seen:

   ```bash
   lsusb               # only the internal hub + Ethernet chip, no dongle
   dmesg | tail -n 20  # the real story
   ```

**What I learned from `dmesg`:** the dongle *was* detected for a moment (a Realtek chip, driver `r8188eu`), then two things went wrong:

- the firmware file `rtlwifi/rtl8188eufw.bin` was **missing** (`error -2`), and
- right after, **`USB disconnect`**, most likely the Pi 1's USB port browning out when the radio powered up.

So: missing firmware *and* not enough power. Two problems that look like one.

> **TIL:** `lsusb` tells you if the hardware is there *now*. `dmesg` tells you what happened when you plugged it in. Read `dmesg` first.

## 2. Attempt two: give the Pi a cable instead

Instead of fighting the dongle, I put a **Wi-Fi range extender with an Ethernet port** next to the Pi. The extender joins the home Wi-Fi and the Pi just sees a normal wired network.

Setting up a TP-Link extender, three ways:

1. **WPS**: press WPS on the router, then within two minutes press WPS/RE on the extender.
2. **TP-Link Tether app**
3. **Browser**: join the extender's own Wi-Fi and open `http://tplinkrepeater.net` (or `192.168.0.254`).

Configure it next to the router, then move it roughly halfway to where you need the signal.

## 3. "The lights are on but nobody's home"

The Ethernet LEDs on the Pi were lit, but the Pi never got an IP address. Things worth checking:

- The extender must act as a **bridge**: the **main router** hands out IP addresses, not the extender.
- Compare subnets: if the Pi and your laptop aren't in the same `192.168.x.` range, they can't talk.
- Plug the Pi **directly into the router** for one minute. If it gets an IP there, the extender is the problem; if not, the Pi is.

In the end the quickest fix was a **fresh SD card image**.

## 4. Fresh install, still "Connection refused"

I flashed a new image with **Raspberry Pi Imager** and used its settings (gear icon) to enable SSH and create my own user. The Pi booted and showed its IP. SSH said `Connection refused`.

**What I learned about debugging SSH, in order:**

1. **`ping` the Pi.** If ping works, the network path is fine and the problem is the SSH service or the address.
2. **`ssh -v user@ip`** shows where the handshake stops.
   - *Connection refused* → nothing is listening on port 22 at that address.
   - *Permission denied* → SSH is fine, credentials aren't.
3. **Check the IP really belongs to the Pi** in the router or extender client list. After re-imaging, it may have a new one.
4. **Clear the old fingerprint** if you've connected to that IP before:

   ```bash
   ssh-keygen -R 192.168.x.x
   ```

5. **Give the first boot a few minutes**, because it generates host keys.
6. If you can't get in at all: put an **empty file named `ssh`** in the boot partition of the SD card, or borrow a keyboard and run `sudo systemctl enable --now ssh`.

Also: you **can't SSH to a MAC address**. You *can* use the MAC address in your router to find, or permanently reserve, the Pi's IP.

Eventually I was in. 🎉

## Takeaways

- On a Pi 1, USB Wi-Fi is a power *and* driver problem. A powered hub plus the right firmware package, or skip it entirely.
- An extender with an Ethernet port is a neat way to bring a wired-only device onto Wi-Fi.
- `dmesg` > guessing.
- For SSH: ping → `ssh -v` → confirm the IP → clear `known_hosts`.

**Next up:** the Python bridge that turns an HTTP request into a relay click.
