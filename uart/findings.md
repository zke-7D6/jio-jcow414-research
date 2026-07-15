# UART Findings

## Setup

- Adapter: CP2102 USB-TTL (3.3V)
- Terminal: minicom, 115200 8N1
- Header: J4 (see [uart-pinout.md](../hardware/uart-pinout.md))

## Boot Log

Full boot log available: [router_bootlog.txt](router_bootlog.txt)

## Key Findings from Boot Log

### System Summary

```
Kernel:       Linux 3.18.21
Build:        Tue Oct 17 15:52:28 IST 2023
Builder:      root@tn795
Toolchain:    gcc 4.6.3 (Buildroot 2015.08.1)
CPU:          MIPS 1004Kc @ 900MHz (4 cores)
RAM:          512MB DDR3 (432MB usable)
Filesystem:   SquashFS (read-only rootfs)
Config store: UBIFS on NAND
```

### Boot Sequence

```
1. DDR3 init (512MB detected)
2. SPI NAND probe (Micron MT29F2G01, 256MB)
3. "Press 'x' or 'b' key in 1 secs" → bootloader upgrade prompt
4. "Press any key in 1 secs" → boot command mode prompt
5. Kernel decompresses (LZMA)
6. SquashFS root mounted read-only
7. UBI volumes attached (/flash and /flash2)
8. Services start (web, VoIP, TR-069, EasyMesh, GPON)
9. "Solution for RIL"
10. "reliance.reliance login:"
```

### Two Locked Prompts

Both prompts during boot require authentication:

**Prompt 1 — Bootloader upgrade console:**
```
Press 'x' or 'b' key in 1 secs to enter or skip bootloader upgrade.
Username:
Password:
```
Behavior: Loops immediately on wrong credentials. No timeout, no error message, no fallback to U-Boot.

**Prompt 2 — Boot command mode:**
```
Press any key in 1 secs to enter boot command mode.
Username:
Password:
```
Same behavior as Prompt 1.

**Linux shell (end of boot):**
```
Solution for RIL
reliance.reliance login: root
Password:
Login incorrect
```
Username `root` confirmed valid. Password is unknown — set at build time as a random string.

### Notable Boot Log Lines

```
Booting RIL...
```
Confirms RIL (Reliance Industries Limited) firmware layer.

```
VFS: Mounted root (squashfs filesystem) readonly on device 31:3
```
Root filesystem is read-only SquashFS. Standard JFC telnet exploit requires writing
to `/flash` — only works on firmware versions where the filesystem allows it (<=R2.3 confirmed).

```
Thirdparty configuration exists
```
Router detects and loads a third-party config at boot. Source and content unknown.

```
Starting smartcableONTApp
wan Mac Id is F0EDB8B4C214
tr69 Integration starts
correlator Id is 1530057737
Response from tr69client:145
```
TR-069 management client starts and contacts Jio's backend. This is how Jio manages
the device remotely and pushes firmware updates.

### Running Services at Boot

- Web server
- VoIP / SIP (le9641 SLIC chip)
- EasyMesh daemon (Wi-Fi mesh)
- GPON/xPON fiber driver
- NTP daemon
- Firewall daemon
- DHCP server
- TR-069 client (smartcableONTApp)
- captivePortal daemon
- NSCAN daemon
- DMS (media server)

### Console Noise Issue

System daemons write to the same TTY as the login prompt, causing daemon output
to be injected into authentication inputs during and after boot.

Example observed:
```
reliance.reliance login: root
Password: Starting QOS daemon... Done
Login incorrect
```

This is a unique characteristic of R2.52.1 that makes UART-based credential
testing unreliable.

## Access Attempts

Username `root` confirmed valid on the Linux shell prompt. Common default
credentials attempted manually — all failed. Password is confirmed to be a
non-standard random string per community research on this platform.

## Next Steps

1. **Firmware downgrade to R2.15** via web UI — enables JFC telnet exploit
2. **CH341A + SOIC-8 clip** — direct SPI NAND read, extract `/etc/shadow`, crack hash offline
3. **Web UI fuzzing** — full directory scan may reveal hidden endpoints
