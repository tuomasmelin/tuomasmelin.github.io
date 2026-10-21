---
title: "TIL: Push the orange tab first (Garage Door Project, Part 5/5)"
date: 2026-10-20
description: 'Connecting the real garage door opener, putting a button on my phone, and making it reachable from outside the house.'
series: "Opening my garage door with my phone"
tags: [home-assistant, wiring, tailscale, ssh, til]
---

**Series:** Opening my garage door with my phone — Part 5 of 5
**Learned:** 15 July – 20 October 2026

Everything worked on the desk. Home Assistant was running on the old laptop. Time to connect the real garage door opener, put a button on my phone, and make it reachable from outside the house.

## 1. Which wire goes into a garage door opener?

My opener has a 6-pin green **spring-clamp terminal block** with little orange tabs. My jumper wires were too thin to stay put.

**What I learned:**

- **Jumper wires are for pin headers**, not for spring terminals.
- Use **solid-core** copper, roughly **0.5–0.75 mm²**: alarm or doorbell cable, a pair from an old Cat5/Cat6 cable, or a single conductor stripped out of ordinary installation cable (e.g. MMJ, usually 1.5 mm², which still fits).
- Stranded wire wants a **ferrule** (pääteholkki) crimped on; loose strands splay out and slip.
- Strip about **8–10 mm**. Only copper should be inside the clamp, no insulation.

## 2. Push the orange tab first

The morning after my first successful test, the wires had **slipped half out** of the opener.

**What I learned:** on a spring-clamp terminal you **press the orange tab down first** (small screwdriver or pen tip), push the wire in, *then* release the tab. I had just been shoving wires into the holes, so the spring never clamped. A gentle tug test tells you immediately.

## 3. Find the right terminals, and test safely

- **The manual or the label on the opener is the authority** for which two terminals are the push-button/impulse input. I got two different answers from an AI on this (first 3 & 4, later 1 & 2), so I found the opener's PDF manual online. On my unit it's **terminals 1 and 2**. Lesson: when an AI gives you two answers, the manufacturer's documentation decides.
- **Mains voltage lives inside that housing.** Unplug the opener before poking at its terminals.
- **Clear the door's path** before any test.
- To find the input safely: put a short wire in each candidate terminal and **touch the free ends together for a split second**, like a button press. Don't leave them shorted, or the opener sees a button held down.
- Keep the remote in hand in case the door does something unexpected.

## 4. COM and NO, not NC

On the relay's three screw terminals the **middle one is COM**. I had put a wire into **NC** (normally closed), which would mean "button held down" whenever the relay is *off*. Wires go to **COM + NO**. Polarity doesn't matter on the opener side.

## 5. Home Assistant: a button and a health check

In `configuration.yaml`:

{% raw %}
```yaml
rest_command:
  toggle_garage:
    url: "http://<pi-ip>:5000/api/toggle"
    method: "POST"

rest:
  - resource: "http://<pi-ip>:5000/api/ping"
    scan_interval: 30
    sensor:
      - name: "Garage API Status"
        unique_id: garage_api_status
        value_template: "{{ value_json.status }}"
```
{% endraw %}

**What I learned:**

- `found character '\t' that cannot start any token` → **YAML forbids tabs**. Spaces only.
- REST commands can be reloaded from *Developer Tools → YAML*; the `rest` sensor needs a restart.
- Test from *Developer Tools → Actions* with `rest_command.toggle_garage` before building UI.
- Dashboard: *Edit dashboard → Add card → Button*, tap action **Perform action** → `rest_command.toggle_garage`. (In my Finnish UI: *Muokkaa kojelautaa → Lisää kortti → Painike*.)
- For a **desktop widget or phone shortcut**, create an automation with a **Webhook trigger** that calls the same action. Anything that can send an HTTP POST to `/api/webhook/<id>` can now open the door.

## 6. Months later: "my Pi speaks ancient SSH?"

After the Pi had sat untouched for a while, SSH greeted me with a cascade of errors:

```
no matching key exchange method found ... diffie-hellman-group1-sha1
no matching host key type found. Their offer: ssh-dss
no matching cipher found. Their offer: aes128-cbc,...
Host key for 192.168.x.50 has changed
```

…and after forcing all the legacy options, `Connection closed by remote host`.

**What I learned:** check the router's DHCP client list *before* fighting SSH. The Pi's **IP had changed**, and a completely different (and much older) device had taken over the old address. All those legacy crypto errors came from *that* device, not my Pi. With the Pi's new IP, SSH just worked, no legacy flags needed. If SSH suddenly starts complaining about ancient algorithms, first ask: *am I even talking to the right machine?* Two fixes:

- **DHCP reservation** in the router: tie the Pi's MAC address to a fixed IP.
- A health check like `/api/ping` + `systemctl status garage.service` tells you within seconds whether everything survived.

## 7. Reaching it from outside: Tailscale

To open the door when I'm not on the home Wi-Fi, I went with **Tailscale** instead of opening ports on the router.

**What I learned:**

- Guides that say *Settings → Add-ons → Tailscale* assume **Home Assistant OS**. With Home Assistant running in **Docker** there is no add-on store, so Tailscale goes on the **host** (the antiX laptop) instead.
- Private access (only devices logged into my tailnet) is the safer default. Funnel exposes Home Assistant publicly; behind a proxy, HA also needs `use_x_forwarded_for` and `trusted_proxies`.
- On antiX, `tailscale up` failed with `failed to connect to local tailscaled; it doesn't appear to be running`. Install scripts assume systemd, which starts `tailscaled` for you; on antiX's init it doesn't happen. Start it yourself, then log in:

  ```bash
  sudo tailscaled &
  sudo tailscale up
  ```

  To survive a reboot, `tailscaled` has to be added to antiX's own service management.

- In the Home Assistant Companion app, set the **External URL** to the laptop's Tailscale address (`http://100.x.x.x:8123`).

**And it worked.** With `tailscaled` running on the laptop and the Tailscale VPN switched on in my phone, I could open Home Assistant from outside the house and tap the garage button. No ports opened on the router, and nothing exposed to the public internet. The one thing to remember is that the **Tailscale VPN has to be connected on the phone**; without it, the 100.x address goes nowhere.

## The whole journey in one picture

```
Phone (HA app, via Tailscale)
  → antiX laptop running Home Assistant in Docker
    → REST call over Wi-Fi
      → Raspberry Pi 1 (Flask, behind a Wi-Fi extender's Ethernet port)
        → USB serial
          → Arduino Uno
            → Keyes relay (COM + NO)
              → garage door opener push-button terminals
```

## What I'd tell myself on day one

1. A garage opener just needs two wires touched together. The electronics are the easy part.
2. **Networking ate most of the time**: old Pi, old BIOS, old SSH, changing IPs. Reserve IPs early.
3. Test every link of the chain in isolation.
4. `dmesg`, `ssh -v` and the router's client list answer most "why doesn't it work" questions.
5. Not everything is systemd. Not everything is Home Assistant OS. Read guides with that in mind.
6. For remote access, a private VPN like Tailscale beats opening router ports. It took one command on the server and one toggle on the phone.
7. Push the orange tab first.

From "can I make my garage door talk to an API?" in early July to opening it from my phone over Tailscale in October: it works. 🚗🚪
