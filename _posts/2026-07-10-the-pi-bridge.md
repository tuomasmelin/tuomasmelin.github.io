---
title: "TIL: Turning an HTTP request into a relay click (Garage Door Project, Part 3/5)"
date: 2026-07-10
description: 'With the Arduino clicking on command and the Raspberry Pi finally reachable, it was time to connect them with a small Flask API.'
series: "Opening my garage door with my phone"
tags: [raspberry-pi, python, flask, systemd, serial, til]
---

**Series:** Opening my garage door with my phone — Part 3 of 5
**Learned:** July 2026

With the Arduino clicking on command (Part 1) and the Raspberry Pi finally reachable (Part 2), it was time to connect them:

```
Laptop/phone → Wi-Fi → extender → Pi (Flask API) → USB serial → Arduino → relay
```

## 1. Find the Arduino on the Pi

Plug the Arduino into the Pi with a USB **data** cable, then:

```bash
ls /dev/ttyUSB* /dev/ttyACM*   # clones often show up as ttyUSB0
dmesg | tail -n 20             # look for "ch341-uart ... attached to ttyUSB0"
lsusb                          # is it on the bus at all?
```

Let your user access serial ports without sudo (log out/in or reboot afterwards):

```bash
sudo usermod -a -G dialout $USER
```

## 2. Test the serial link by hand

Before writing any Python, talk to the Arduino directly:

```bash
sudo apt install screen
screen /dev/ttyUSB0 9600
# type o → relay clicks
# exit with Ctrl+A, then \
```

## 3. The bridge: Flask + pyserial

```python
import time
import serial
from flask import Flask, jsonify

SERIAL_PORT = "/dev/ttyUSB0"
ser = serial.Serial(SERIAL_PORT, 9600, timeout=1)
time.sleep(2)  # the Arduino resets when the port opens

app = Flask(__name__)

@app.route("/api/toggle", methods=["POST"])
def toggle():
    ser.write(b"o")
    return jsonify({"status": "triggered"})

@app.route("/api/ping", methods=["GET"])
def ping():
    return jsonify({"status": "ok"})

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

The `/api/ping` endpoint turned out to be very handy later for health checks in Home Assistant.

## 4. Make it a service

Running the script by hand doesn't survive a reboot, so it became a systemd unit at `/etc/systemd/system/garage.service`:

```ini
[Unit]
Description=Garage Door API
After=network.target

[Service]
ExecStart=/usr/bin/python3 /home/<user>/garage/garage_server.py
WorkingDirectory=/home/<user>/garage
Restart=always
User=<user>

[Install]
WantedBy=multi-user.target
```

My first `systemctl start` failed with `Unit garage.service not found`.

**What I learned:** systemd doesn't watch that folder. After creating or editing a unit file you must run:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now garage.service
sudo systemctl status garage.service
sudo journalctl -u garage.service -n 50 --no-pager   # when something's off
```

## 5. The serial port can only have one owner

The service started, but couldn't talk to the Arduino. The culprit was my **forgotten `screen` session** from step 2, still holding `/dev/ttyUSB0`.

```bash
screen -ls                        # list sessions
screen -X -S <session_id> quit    # kill one
sudo fuser /dev/ttyUSB0           # who is using the port?
sudo fuser -k /dev/ttyUSB0        # make them stop
sudo systemctl restart garage.service
```

> **TIL:** a serial port is exclusive. Serial Monitor, `screen`, a stray Node script: any of them will silently block your service.

## 6. The first real success

From my laptop:

```bash
curl -X POST http://<pi-ip>:5000/api/toggle
```

**Click.** The whole chain worked end to end.

## 7. A word on security

At this point anyone on my Wi-Fi who knows the Pi's IP can "press the button". Easy first improvements:

- Require a secret header (e.g. `X-API-KEY`) and return `401` without it.
- **Don't open router ports** for remote access; use a private tunnel instead (that's Part 5).

## Takeaways

- Test each link of the chain on its own: Serial Monitor → `screen` → `curl`.
- `daemon-reload` after every unit file change.
- Only one process can own a serial port.
- Add a cheap `/ping` endpoint from day one.

**Next up:** I needed a proper home for Home Assistant, which meant reviving a very old laptop.
