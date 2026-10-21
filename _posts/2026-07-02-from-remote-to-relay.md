---
title: "TIL: A garage door button is just two wires touching (Garage Door Project, Part 1/5)"
date: 2026-07-02
description: 'I can open my garage door with a remote. I wanted to open it with an API call instead, and had no idea what electronics sit between "HTTP request" and "door moves".'
series: "Opening my garage door with my phone"
tags: [arduino, relay, electronics, garage-door, til]
---

**Series:** Opening my garage door with my phone — Part 1 of 5
**Learned:** 2–4 July 2026

I can open my garage door with a remote. I wanted to open it with an API call instead. I know how to write the software, but I had no idea what electronics sit between "HTTP request" and "door moves". This post covers the first few days: figuring out the hardware, digging through an old microcontroller kit, and getting a relay to click on my desk.

## 1. The opener only needs a "dry contact"

The wall button on a garage door opener doesn't send any power anywhere. It just **shorts two low-voltage terminals together for a moment**. That's called a *dry contact*.

So the whole "smart" part boils down to: something that can briefly connect two wires when told to. That something is a **relay**.

- Wire the relay's **COM** (common) and **NO** (normally open) terminals *in parallel* with the existing wall button.
- When the relay closes for ~0.5 s, the opener thinks someone pressed the button.
- The existing remotes and wall button keep working. Nothing is replaced, only added.

Optional extra I've parked for later: a **magnetic reed switch** on the door frame to know whether the door is actually open or closed.

## 2. Inventory what you already have

I had an old kit from Hobby Components lying around. Relevant finds:

- **Arduino Uno** (clone)
- **Keyes 5V relay module** with three pins: `S` (signal), `+` (5V) and `-` (GND), plus three screw terminals: NC, COM, NO
- **Arduino Prototype Shield v5**, a board that stacks on the Uno for soldering your own circuit onto

**What I learned:** the Uno has **no networking at all**. To expose an API you need *something* with Wi-Fi or Ethernet. The options were:

| Option | Notes |
| --- | --- |
| ESP8266 (ESP-01) module next to the Uno | Runs at 3.3V, talks to the Uno over serial with AT commands. Fiddly. |
| Replace the Uno with an ESP32 / Wemos D1 | Built-in Wi-Fi, can host the API itself. Logic is 3.3V, so a 5V relay may need a level shifter. |
| **Uno + Raspberry Pi over USB** | Pi runs the API in a "real" language, Uno only flips the relay. |

I chose the last one: an old Raspberry Pi as the **brain** and the Arduino as the **muscle**. (Spoiler: the Pi being old became its own adventure, see Part 2.)

## 3. Wiring the relay for a bench test

Before touching the garage at all, test on the desk with the Arduino plugged into the computer.

| Relay pin | Arduino pin |
| --- | --- |
| `+` | 5V |
| `-` | GND |
| `S` | Digital pin 7 |

Leave the COM/NO screw terminals **empty** during the bench test.

**What I learned:** many of these cheap relay modules are **active LOW**: writing `LOW` to the signal pin turns the relay *on*. So the sketch starts by writing `HIGH` to keep it off.

```cpp
const int relayPin = 7;

void setup() {
  pinMode(relayPin, OUTPUT);
  digitalWrite(relayPin, HIGH);   // relay off (active LOW module)
  Serial.begin(9600);
}

void loop() {
  if (Serial.available() > 0) {
    char c = Serial.read();
    if (c == 'o' || c == 'O') {
      digitalWrite(relayPin, LOW);  // "press the button"
      delay(500);
      digitalWrite(relayPin, HIGH); // "release"
    }
  }
}
```

Open the Serial Monitor at 9600 baud, type `o`, press Enter → **click, click**. The little LED on the relay blinks with it.

Bonus check: put a multimeter in continuity mode across COM and NO. It should beep for exactly the half second the relay is closed.

## 4. The detour: Johnny-Five and Firmata

My first instinct as a JavaScript developer was to drive the board from Node.js with **Johnny-Five**. It timed out.

**What I learned:** Johnny-Five needs the **StandardFirmata** sketch uploaded to the board first (`File → Examples → Firmata → StandardFirmata`, the plain one, not the Plus version). Without it the computer and the board don't speak the same protocol. In the end I didn't need Johnny-Five at all, because a ten-line sketch listening on serial was simpler.

## 5. "programmer is not responding"

Uploading anything to the Uno failed with:

```
avrdude: stk500_recv(): programmer is not responding
avrdude: stk500_getsync() attempt 1 of 10: not in sync: resp=0x00
```

This means the computer is shouting down the cable and the board isn't answering. The checklist that helped:

- **Pick the right port.** On a Mac, ignore `Bluetooth-Incoming-Port`; look for `/dev/cu.usbmodem…` (or `/dev/cu.wchusbserial…` on clones).
- **Use a data cable**, not a charge-only one. Lights turning on prove nothing.
- **Take the shield off** and upload to the bare Uno first. A stacked shield can interfere with the serial pins.
- **Close anything else holding the port**: the Serial Monitor, or a Node script still running in another terminal.
- **Clones may need the CH340 driver.**

## 6. Soldering is optional (for now)

The Arduino pins are small and I was nervous about bridging two of them with solder. Alternatives that are perfectly fine for this phase:

- **Female-to-female / male-to-female jumper wires** straight onto the header pins
- A **solderless breadboard** (great for testing, less great long-term in a cold garage)
- A **screw terminal shield**: strip the wire, tighten a screw, done

## Takeaways

- A garage door opener's button input is a dry contact. A relay across COM/NO is all it takes.
- An Arduino Uno can't do networking on its own; decide early what provides the network.
- Cheap relay modules are often active LOW.
- Bench test first: Serial Monitor + listening for the click + a multimeter on COM/NO.
- "programmer is not responding" is almost always port, cable, shield or a busy serial port.

**Next up:** getting a first-generation Raspberry Pi onto my network, which took far longer than the electronics.
