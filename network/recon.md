# Network Reconnaissance

## Target

```
Device:   Jio Fiber JCOW414
LAN IP:   192.168.29.1
```

## Port Scan Results

Full TCP range scanned with nmap.

### Open Ports

| Port | Protocol | Service | Notes |
|---|---|---|---|
| 53 | TCP/UDP | DNS | Router DNS resolver |
| 80 | TCP | HTTP | Web admin UI |
| 443 | TCP | HTTPS | Web admin UI |

### Closed / Not Present

| Port | Service |
|---|---|
| 21 | FTP |
| 22 | SSH |
| 23 | Telnet |
| 69 | TFTP |
| 161 | SNMP |
| 7547 | TR-069 (not confirmed on LAN side) |
| 8080 | Alt HTTP |

No shell access exposed on the LAN interface. Jio intentionally locks down the attack surface — remote management is handled via TR-069 from Jio's backend servers outbound, not inbound.

## Web UI Enumeration

### Standard Pages

Full admin access obtained with default credentials. Features available:
- Port Forwarding
- DMZ
- UPnP
- Static / Classical Routing
- User Management (Admin and Guest roles only)
- Backup / Restore
- Firmware Upgrade (requires image + signature file)

Bridge mode is **not available** in the web UI on this firmware version.

### Hidden Endpoints

Manually tested URL paths:

| URL | Response | Notes |
|---|---|---|
| /info.html | **403** | Page exists, access blocked |
| /status.html | **403** | Page exists, access blocked |
| /debug | 404 | Not found |
| /cgi-bin/ | 404 | Not found |
| /adm/ | 404 | Not found |
| /shell | 404 | Not found |
| /diag | 404 | Not found |

The 403 responses on `/info.html` and `/status.html` confirm these pages exist but are restricted. Content unknown — possible diagnostic or status pages.

## TR-069 Observation

The boot log confirms a TR-069 client (`smartcableONTApp`) runs on the router and connects outbound to Jio's management servers. This is how Jio pushes firmware updates and config changes remotely. The client is not directly accessible from the LAN.

```
correlator Id is 1530057737
Response from tr69client:145
Tr69InterfaceFetchstart success: Boot SendMsg response 0
```

## Conclusion

The LAN-facing attack surface is intentionally minimal. No shell services are exposed. Deeper access requires physical methods (UART or flash programmer) or a firmware downgrade to enable the telnet exploit documented by the JFC community.
