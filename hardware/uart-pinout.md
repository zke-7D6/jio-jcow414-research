# UART Header — J4

## Physical Location

![J4 Header](photos/j4_uart.jpg)

The J4 header is a 4-pin populated header located on the PCB front side:
- Positioned between the large 2200µF capacitor and the metal RF shield
- Silkscreen label "J4" visible, with "4" marked on the right side
- Pins are present (short metal pins, not just pads)

## Pinout

**Fully confirmed by multimeter measurement.**

```
J4 Header (left to right)

┌─────┬─────┬─────┬─────┐
│  1  │  2  │  3  │  4  │
│ GND │ TX  │ RX  │ VCC │
│     │     │     │3.3V │
└─────┴─────┴─────┴─────┘
                    ↑
                    DO NOT
                    CONNECT
```

| Pin | Function | Connect to |
|---|---|---|
| 1 (left) | GND | CP2102 GND |
| 2 | TX (router → PC) | CP2102 RX |
| 3 | RX (PC → router) | CP2102 TX |
| 4 (right) | VCC 3.3V | Nothing — leave floating |

> All four pins confirmed via multimeter continuity and voltage measurement.

## Pin Identification Method

1. **GND** — Continuity mode multimeter. Probe each pin against metal RF shield. The pin that beeps is GND.
2. **VCC** — DC voltage measurement with router powered on. Pin sitting at 3.3V idle = VCC.
3. **TX** — Remaining pin that fluctuates 0–3.3V during boot = TX (router transmitting).
4. **RX** — Remaining pin sitting near 0V idle = RX.

## Connection Settings

```
Baud rate:    115200
Data bits:    8
Parity:       None  
Stop bits:    1
Flow control: None
```

## Hardware Used

- CP2102 USB-TTL adapter (3.3V logic level)
- Female-to-female dupont jumper wires
- Terminal: minicom on Kali Linux

## Console Noise Issue

**Important:** The UART TTY on firmware R2.52.1 is shared between the login process and system daemons. Background processes write directly to the serial console at unpredictable intervals, including during authentication prompts.

This means daemon output gets injected into password inputs:

```
reliance.reliance login: root
Password: Starting QOS daemon... Done
Login incorrect
```

In the example above, the password prompt received the daemon message as part of the input, causing a false "Login incorrect" even if the correct password was being sent.

This makes scripted brute-force attempts unreliable on this firmware version. Manual attempts during quiet windows are more reliable but the noise is frequent enough to make this approach impractical.
