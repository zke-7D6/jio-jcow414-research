# Research Log

## Session 1 — Hardware Identification

- Opened router, photographed PCB front and back
- Identified all major ICs by visual inspection of markings
- Located J4 UART header (4-pin, populated, near large capacitor)
- Confirmed SPI NOR flash at U19 (Puya P25Q80L)
- Identified SK Hynix H5TQ4G63EFR RAM (512MB DDR3)
- EcoNet EN7528HU SoC confirmed (MIPS-based, GPON integrated)
- Two MediaTek Wi-Fi chips: MT7592N and MT7613BEN

## Session 2 — Web UI and Network Recon

- Full admin access at 192.168.29.1
- nmap full TCP scan — only ports 80, 443, 53 open
- No telnet, SSH, FTP, TFTP, SNMP on LAN
- Hidden endpoints: /info.html and /status.html return 403
- Downloaded config backup from Maintenance → Backup/Restore
- Firmware upgrade page requires both .img and .sig files

## Session 3 — Config Backup Analysis

- Identified as OpenSSL AES-256-CBC encrypted
- Attempted decryption with all label values and common passwords
- Tried keyguesser.py from JFC Group — fails on R2.52.1
- Inner layer entropy 7.999/8.0 — double encrypted
- Concluded: not decryptable without firmware downgrade

## Session 4 — UART Access

- Purchased CP2102 USB-TTL adapter and multimeter
- Identified J4 pins using continuity and voltage measurement
- Connected CP2102 at 115200 8N1, opened minicom
- Successfully captured full boot log on first attempt
- Discovered two separate locked prompts during boot
- Linux shell confirmed at end of boot: reliance.reliance login:
- Username root confirmed valid, password unknown
- Discovered console noise issue — daemons share TTY with login

## Key Discoveries

- Router manufactured by Sercomm, customized by RIL (Reliance)
- Internal EcoNet codename: TC3162 / EN751221
- Build hostname: root@tn795
- Firmware R2.52.1 has read-only filesystem (blocks standard JFC exploit)
- Two separate boot prompt windows, both locked
- Console TTY shared with system daemons — unique to newer firmware
- "Thirdparty configuration exists" logged at boot — unknown significance
- TR-069 client (smartcableONTApp) contacts Jio backend on startup
- Dual firmware design: tclinux (primary) + tclinux_slave (backup)

## Pending

- [ ] Firmware downgrade to R2.15 via web UI
- [ ] Full directory fuzz of web UI with gobuster
- [ ] CH341A + SOIC-8 clip for direct flash read
- [ ] Extract /etc/shadow from SquashFS, crack hash with hashcat
- [ ] Document "Thirdparty configuration exists" — identify source
