---
title: "Zyxel Firmware Analysis — Extraction, ZIP Cracking, and Credential Recovery"
date: 2025-01-07
draft: false
description: "Full firmware analysis of a Zyxel network device: binwalk extraction of a squashfs filesystem, known-plaintext attack on the ZIP encryption with pkcrack, and hashcat-based credential recovery from shadow.basic."
tags: ["firmware", "reverse-engineering", "binwalk", "hashcat", "cryptography", "embedded"]
---

> Academic lab — FSI M2, application security module. Target: Zyxel firmware binary (`455ABUJOCO.bin`). Goal: extract the filesystem, identify sensitive data, and recover credentials through cryptographic attacks.

---

## Environment

**OS**: Ubuntu VM
**Tools**: `binwalk`, `pkcrack`, `hashcat`, Python (wordlist generation)
**Target**: `455ABUJOCO.bin` — Zyxel router/firewall firmware

---

## Step 1 — Firmware Extraction

Starting with the raw firmware binary, `binwalk` was used to identify embedded filesystems and compression formats:

```bash
binwalk 455ABUJOCO.bin
```

`binwalk` identified a compressed filesystem signature inside the binary. Extraction:

```bash
binwalk -e 455ABUJOCO.bin
```

The extracted filesystem is a **squashfs** image — a read-only compressed filesystem commonly used in embedded Linux devices. Mounting it reveals a Linux directory structure:

```
/zyxel/
/ftp/
/etc/
/bin/
/usr/
...
```

---

## Step 2 — Configuration File Discovery

Navigating the extracted filesystem, the target file was `/etc/zyxel/sys/system-default.conf` — a 27KB configuration file containing device settings, service configurations, and credentials.

This file appears at the same path within the firmware as it does in the ZIP archive bundled with the firmware — a key observation for the next step.

---

## Step 3 — ZIP Cracking (Known-Plaintext Attack)

The firmware is distributed as a ZIP archive with encrypted contents. The encryption uses a legacy PKZIP stream cipher — vulnerable to a known-plaintext attack if any plaintext content is available.

Since `system-default.conf` was already extracted from the raw binary and exists at a known path within the ZIP, this file is the plaintext. `pkcrack` performs the attack:

```bash
pkcrack -C encrypted_firmware.zip \
        -c "etc/zyxel/sys/system-default.conf" \
        -P plaintext_archive.zip \
        -p "etc/zyxel/sys/system-default.conf" \
        -d decrypted_firmware.zip \
        -a
```

The 5 parameters:
- `-C` — target encrypted ZIP
- `-c` — path of the known file inside the encrypted ZIP
- `-P` — ZIP containing the known plaintext
- `-p` — path of the known file inside the plaintext ZIP
- `-d` — output decrypted ZIP
- `-a` — find all valid keys

`pkcrack` recovered the ZIP encryption keys. The ZIP archive was fully decrypted.

**Why this works**: PKZIP's traditional encryption uses three 32-bit keys initialized from the password. If an attacker has ~13 bytes of known plaintext at a known offset, the keystream can be recovered through a meet-in-the-middle attack. This encryption scheme has been broken since 1994 (Biham & Kocher).

---

## Step 4 — Credential Recovery

Inside the decrypted archive, at `/etc/zyxel/sys/`, two files were found:

- `passwd.basic` — username entries
- `shadow.basic` — hashed passwords (MD5Crypt, `$1$` prefix)

The hash format is `$1$` (MD5Crypt, 500 iterations of MD5). To crack:

1. Generated a targeted wordlist with Python based on known Zyxel password patterns
2. Ran `hashcat` in mode `-m 500` (MD5Crypt):

```bash
hashcat -m 500 shadow.basic wordlist.txt
```

**Result**: password cracked — `Pr0w!aN_fXp`

The crack succeeded in seconds. MD5Crypt with 500 iterations provides minimal resistance against modern GPUs.

---

## Vulnerabilities Summary

### 1. Weak ZIP Encryption (CVE class: Cryptographic Weakness)

The firmware uses PKZIP legacy encryption — a stream cipher broken in 1994. A known-plaintext attack recovers the full archive with one file that exists in both the encrypted ZIP and the raw binary.

**Fix**: Use AES-256 encrypted ZIP (WinZip AES or 7-Zip AES), or sign and encrypt the firmware with asymmetric cryptography.

### 2. Sensitive Credentials in Accessible Configuration Files

`shadow.basic` and `passwd.basic` are stored inside the firmware archive without additional protection beyond the (now broken) ZIP encryption. Once the ZIP is decrypted, credentials are directly accessible.

**Fix**: Do not store plaintext or weakly hashed credentials in firmware images. Use device-specific keys derived at provisioning time.

### 3. Weak Password Hashing

MD5Crypt (`$1$`) is deprecated. 500 iterations provide negligible cost to an attacker with GPU access.

**Fix**: Use bcrypt, scrypt, or Argon2 for password storage.

---

## Attack Chain Summary

```
455ABUJOCO.bin
    │
    ├── binwalk -e → squashfs extraction
    │       └── /etc/zyxel/sys/system-default.conf (27KB)
    │
    ├── pkcrack (known-plaintext on PKZIP)
    │       └── decrypted_firmware.zip
    │               └── shadow.basic, passwd.basic
    │
    └── hashcat -m 500 → Pr0w!aN_fXp
```

---

## What I Learned

- Firmware analysis starts with `binwalk` — it identifies compression and filesystem signatures regardless of the file extension
- PKZIP legacy encryption is fundamentally broken — one known file inside the archive is enough to recover all keys
- MD5Crypt is not a secure password hashing scheme for modern threat models — cost is too low
- Configuration files in firmware often contain more than expected — default credentials, service tokens, network configuration
- The attack chain (extract → identify overlap → known-plaintext → hash crack) is entirely reproducible with public tools

---

## Resources

- [binwalk](https://github.com/ReFirmLabs/binwalk)
- [pkcrack](https://github.com/keyunluo/pkcrack)
- [hashcat](https://hashcat.net/hashcat/)
- [Biham & Kocher — A Known Plaintext Attack on the PKZIP Stream Cipher (1994)](https://link.springer.com/chapter/10.1007/3-540-60590-8_12)
- [MD5Crypt weakness analysis](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf)
