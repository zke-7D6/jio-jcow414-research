# Config Backup Encryption Analysis

## File

```
Filename:  RSVOTAJEM210410_JCOW414.enc
Size:      147KB
Source:    Downloaded from web UI → Maintenance → Backup/Restore
```

The filename contains the router's RSN (RSVOTAJEM210410) and model (JCOW414).

## Encryption Layer 1

```bash
file RSVOTAJEM210410_JCOW414.enc
# Output: openssl enc'd data with salted password
```

Confirmed **OpenSSL AES-256-CBC** encryption with salt (`Salted__` header present).

Decryption attempt using common passwords, label values (MAC, EAN, RSN, WiFi password), and combined variants — all failed to produce meaningful output.

The JFC community tool `keyguesser.py` uses permutations of router serial + SSID suffix + hardcoded key strings to find the encryption key. **This tool fails on firmware R2.52.1** — it was designed for ≤R2.49.

## Encryption Layer 2

After attempting decryption with the EAN (`8908014230029`) using AES-256-CBC with MD5 key derivation — the output was technically valid but showed near-perfect entropy (7.999/8.0), indicating the inner content is itself encrypted or this was a false positive due to lucky padding.

Byte frequency analysis confirmed uniform distribution — characteristic of encrypted data, not compressed plaintext.

## Conclusion

The config backup appears to be double-encrypted:
- **Outer layer:** OpenSSL AES-256-CBC, key derived from device-specific values
- **Inner layer:** Unknown cipher, possibly server-side key held by Jio

This file is **not practically decryptable** on firmware R2.52.1 without either:
1. Jio's private key (server-side, not on device)
2. Firmware downgrade to R2.15 where `keyguesser.py` works

## Label Values (for reference)

| Field | Value |
|---|---|
| RSN | RSVOTAJEM210410 |
| MAC | F0EDB8B4C214 |
| EAN | 8908014230029 |

## Tools Used

- `file` — initial identification
- `openssl enc` — decryption attempts
- Python entropy analysis — byte frequency distribution
- `keyguesser.py` (JFC Group) — key permutation attempt
