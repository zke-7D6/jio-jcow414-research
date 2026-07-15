# Jio Fiber JCOW414 — Hardware Research

Personal research project on an owned, spare Jio Fiber JCOW414 router.  
Goals: understand internals, gain UART/shell access, explore firmware structure, and configure as a Wi-Fi access point.

> **Status: In Progress** — UART access achieved, shell login blocked. See [Current Status](#current-status).

---

## Device Overview

| Field | Value |
|---|---|
| Model | JCOW414 |
| Hardware OEM | Sercomm Corporation (Taiwan) |
| Firmware | SRCMTFI_JCOW414_R2.52.1 |
| Firmware Date | October 17, 2023 |
| Build Host | root@tn795 |
| Kernel | Linux 3.18.21 (MIPS) |
| SoC | EcoNet EN7528HU (internal codename: TC3162 / EN751221) |
| CPU | MIPS 1004Kc @ 900MHz, 4 cores |
| Wi-Fi (2.4GHz) | MediaTek MT7592N |
| Wi-Fi (5GHz) | MediaTek MT7613BEN |
| RAM | SK Hynix H5TQ4G63EFR — 512MB DDR3 |
| SPI NOR Flash | Puya P25Q80L — 1MB (bootloader only), PCB ref U19 |
| SPI NAND Flash | Micron MT29F2G01 — 256MB (firmware + config) |
| Fiber Connector | SC/APC (GPON) |
| PCB Marking | ASKDCB / E239218 |

---

## Firmware Supply Chain

This router is manufactured by **Sercomm Corporation** and customized by **Reliance Industries Limited (RIL)** for Jio Fiber deployment.

```
Sercomm Corporation  →  Hardware OEM, firmware base
EcoNet (MediaTek)    →  SoC platform, bootloader (TC3162/EN7528)
RIL / Jio            →  Firmware customization layer ("Booting RIL...")
```

Evidence from boot log:
- `Booting RIL...` — RIL firmware layer
- `Solution for RIL` / `reliance.reliance login:` — hostname confirms Reliance branding
- `smartcableONTApp` — Sercomm's proprietary TR-069 management application
- `Thirdparty configuration exists` — unknown third party config detected at boot

---

## Hardware Identification

### Main SoC — EcoNet EN7528HU
- MIPS-based, handles CPU + fiber uplink (GPON)
- Internal EcoNet codename: **TC3162** / EN751221
- Package: LQFP (~200 pins)
- PCB reference: U1
- Bootloader string: `EN7528 at Mon Sep 2 14:09:30 IST 2019 version 1.1`

### Wi-Fi — MediaTek MT7592N
- Dual-band 802.11n/ac (2.4GHz + 5GHz)
- PCB reference: U12
- Date code: 2238-BZPJL

### Wi-Fi — MediaTek MT7613BEN
- 802.11ac + Bluetooth combo (5GHz secondary radio)
- Date code: 2237-BZAEL / AEX10074 / BAP3Y540

### RAM — SK Hynix H5TQ4G63EFR
- DDR3, 512MB (4Gbit)
- PCB reference: U2
- Date code: ADC 207V

### SPI NOR Flash — Puya P25Q80L
- 8Mbit (1MB), SOP-8 package
- PCB reference: U19
- Contains: bootloader only
- Readable with: CH341A programmer + SOIC-8 clip

### SPI NAND Flash — Micron MT29F2G01
- 256MB (2Gbit)
- Contains: full firmware, config partitions
- Detected in boot log: `mfr_id=0x2c, dev_id=0x24`

---

## Flash Partition Layout

From boot log (MTD partition table):

| Partition | Offset | Size | Contents |
|---|---|---|---|
| bootloader | 0x000000000000 | 256KB | U-Boot variant |
| romfile | 0x000000040000 | 256KB | ROM config |
| kernel | 0x000000080000 | ~4MB | Primary kernel |
| rootfs | 0x00000047379c | ~36MB | SquashFS root (read-only) |
| tclinux | 0x000000080000 | 40MB | Primary firmware image |
| tclinux_slave | 0x000002880000 | 40MB | Backup firmware image |
| opt0 | 0x000005080000 | 8MB | Optional partition |
| opt1 | 0x000005880000 | 8MB | Optional partition |
| ubifs | 0x000006080000 | 26MB | Config data (/flash) |
| config | 0x000007a80000 | 2MB | Config partition |
| ubifs2 | 0x000007c80000 | 90MB | Config data 2 (/flash2) |
| reservearea | 0x00000ddc0000 | ~2MB | Reserved |

Key observations:
- Dual firmware design (tclinux + tclinux_slave) — failsafe system
- Root filesystem is **SquashFS, read-only** — confirmed in boot log
- Persistent storage on `/flash` (ubi0) and `/flash2` (ubi1)
- `/flash` = 26MB UBIFS, `/flash2` = 90MB UBIFS

---

## UART Header — J4

### Location
4-pin populated header visible on PCB front side, between the large 2200µF capacitor and the metal RF shield. Silkscreen label "J4" with pin 4 marked on right side.

### Confirmed Pinout
| Pin | Function | Notes |
|---|---|---|
| 1 (left) | VCC 3.3V | Do NOT connect |
| 2 | GND | Connect to adapter GND |
| 3 | TX | Router transmits → adapter RX |
| 4 (right) | RX | Adapter TX → router receives |

### Connection Settings
```
Baud rate:    115200
Data bits:    8
Parity:       None
Stop bits:    1
Flow control: None
```

### Hardware Used
- CP2102 USB-TTL adapter (3.3V logic)
- Female-to-female jumper wires

### Console Noise Issue
The UART TTY is **shared between the login process and system daemons**. Background processes inject output directly into the serial console during and after boot, including during password prompts. This makes scripted login attempts unreliable as daemon output corrupts input.

Example observed:
```
Password: Starting QOS daemon... Done
```

This is a significant finding for anyone attempting UART-based access on R2.52.1.

---

## Boot Sequence

Two separate prompt windows appear during boot:

```
1. "Press 'x' or 'b' key in 1 secs to enter or skip bootloader upgrade."
   → Leads to locked bootloader console (Username/Password)

2. "Press any key in 1 secs to enter boot command mode."
   → Leads to second locked console (Username/Password)
```

Both prompts require authentication. Neither times out or falls through to U-Boot on failed attempts — the router simply loops back to the username prompt indefinitely.

After full boot:
```
Solution for RIL
reliance.reliance login:
```
This is the Linux shell login. Username `root` confirmed valid. Password unknown.

---

## Access Attempts

### Web UI
- Full admin access obtained at `192.168.29.1`
- Features available: Port Forwarding, DMZ, UPnP, Static Routing, User Management, Backup/Restore, Firmware Upgrade
- Bridge mode: unavailable in UI

### Network Reconnaissance

| Port | Service | Status |
|---|---|---|
| 80 | HTTP | Open — web admin UI |
| 443 | HTTPS | Open — web admin UI |
| 53 | DNS | Open — router DNS |
| 22 | SSH | Closed |
| 23 | Telnet | Closed |
| 21 | FTP | Closed |
| 69 | TFTP | Closed |
| 161 | SNMP | Closed |

Hidden web endpoints found:

| URL | Response |
|---|---|
| /info.html | 403 — page exists, access blocked |
| /status.html | 403 — page exists, access blocked |
| /debug, /cgi-bin/, /adm/, /shell, /diag | 404 |

### UART — Bootloader
Both boot prompt windows locked with username/password. No timeout, no fallback, no error message on wrong credentials. Common default credentials attempted — all failed.

### UART — Linux Shell
Username `root` confirmed valid. Password is a random alphanumeric string set at build time — not recoverable via common wordlists. Console noise from daemon output makes scripted attempts unreliable.

### Config Backup Analysis
Downloaded encrypted config backup (`RSVOTAJEM210410_JCOW414.enc`):
- Confirmed OpenSSL AES-256-CBC encrypted (`Salted__` header present)
- Inner layer entropy: 7.999/8.0 — double encrypted or server-side key
- JFC community tool (`keyguesser.py`) fails on firmware R2.52.1 — designed for ≤R2.49
- Likely requires firmware downgrade to R2.15 for decryption

---

## Current Status

| Area | Status | Notes |
|---|---|---|
| Hardware identification | Complete | All ICs identified and documented |
| PCB photography | Complete | All major components photographed |
| Web UI recon | Complete | Full admin access, no shell exposed |
| Network recon | Complete | No shell services on LAN |
| UART connection | Complete | CP2102 connected, boot log captured |
| Boot log analysis | Complete | Full partition map, services documented |
| Config backup analysis | Partial | Double encrypted, key unknown for R2.52.1 |
| Bootloader access | Blocked | Both prompts locked |
| Linux shell access | Blocked | Root password unknown |
| Firmware dump | Pending | Requires shell or flash reader |

---

## Remaining Paths

### Path 1 — Firmware Downgrade (Free)
Downgrade to R2.15 via web UI firmware upgrade page. On older firmware:
- `keyguesser.py` can decrypt config backup → reveals credentials
- JFC telnet exploit works (confirmed by community on ≤R2.3)
- Telnet login: `root` / `Password` (set by jailbreak init script)

Blocker: Requires R2.15 firmware image + signature file from Jio's OTA servers.

### Path 2 — CH341A Flash Read (₹400)
Read SPI NAND flash directly with CH341A programmer + SOIC-8 clip:
- Extract SquashFS from `tclinux` partition
- Get `/etc/shadow` password hash
- Crack offline with hashcat

This bypasses all authentication entirely.

### Path 3 — Web UI Fuzzing
Hidden CGI endpoints may exist behind the 403 responses. A full directory fuzz with Gobuster against `/cgi-bin/` may reveal command execution endpoints.

---

## Community Resources

- [JFC Group — JioFiber Customisation](https://github.com/JFC-Group/JF-Customisation) — root access method, keyguesser tool
- [fawazahmed0/Jio-fiber-Modem](https://github.com/fawazahmed0/Jio-fiber-Modem) — related Jio router research
- [NaniBot — JIO Notes](https://nanibot.net/posts/jio-notes/) — persistent root method documentation
- [Yash Garg — JioFiber Bits](https://yashgarg.dev/notes/jiofiber-bits/) — root access guide

---

## Disclaimer

This research is conducted on personally owned hardware, disconnected from any live network. For educational purposes only. No Jio infrastructure was accessed or targeted at any point.
