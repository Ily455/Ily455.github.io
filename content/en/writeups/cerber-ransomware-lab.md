---
title: "Cerber Ransomware — Static & Dynamic Analysis Lab"
date: 2026-03-06
draft: false
description: "A full walkthrough of analyzing the Cerber ransomware family using Flare-VM and Cuckoo Sandbox — static PE analysis, behavioral tracing, and key indicators of compromise."
tags: ["malware-analysis", "ransomware", "reverse-engineering", "flare-vm", "cuckoo"]
---

> Work done during my internship at **Techso Group** (Casablanca, July–August 2023). I was tasked with studying malware analysis methodologies and performing a hands-on analysis of the Cerber ransomware family under Flare-VM and Cuckoo Sandbox.

---

## What is Cerber

Cerber is a ransomware-as-a-service (RaaS) family, active primarily between 2016 and 2017. It was one of the first to use a text-to-speech engine to announce infection to the victim. Its affiliate model made it massively distributed — operators could license the ransomware and keep a percentage of ransom payments.

Despite its age, it's a relevant learning target. The techniques it uses — API obfuscation, process injection, hybrid encryption, C2 over Tor — are still in use in modern ransomware families. The goal of this lab wasn't to find new things about Cerber, but to build and practice a repeatable analysis workflow.

---

## Lab Setup

### Static analysis — Flare-VM

[Flare-VM](https://github.com/mandiant/flare-vm) is a Windows-based malware analysis distribution maintained by Mandiant. Setup:

1. Fresh Windows 10 VM — **isolated, host-only adapter, no internet access**
2. Disable Windows Defender and automatic updates before running the installer
3. Run the Flare-VM install script — it pulls tools via Chocolatey
4. **Snapshot the clean state** before touching any sample

Key tools used:

| Tool | Purpose |
|------|---------|
| Detect-It-Easy (DIE) | Packer/compiler identification |
| PEStudio | PE header, imports, strings, entropy |
| Ghidra | Disassembly and decompilation |
| x64dbg | Dynamic debugging |
| FLOSS | Obfuscated string extraction |
| CFF Explorer | PE structure viewer |

> 📷 *[Placeholder — Flare-VM desktop]*

### Dynamic analysis — Cuckoo Sandbox

Cuckoo runs on a Linux host and spins up isolated Windows guest VMs to execute samples automatically.

```
Ubuntu host (Cuckoo) → KVM → Windows 7 guest (Cuckoo agent)
```

Key config points:
- Result server on a host interface reachable from the guest
- `machinery = kvm` in `cuckoo.conf`
- Network routed through InetSim (fake internet) — the sample gets responses but no real connectivity
- Snapshot taken after agent installation

```bash
cuckoo submit --timeout 120 cerber_sample.exe
```

---

## Static Analysis

### 1. Initial triage

First pass with DIE before running anything:

```
File type  : PE32 executable (GUI) Intel 80386
Packer     : UPX (some samples) / custom packer
Compiler   : MSVC
Entropy    : High in code section — suggests packed/encrypted payload
```

The high entropy in the code section is the first indicator — legitimate software rarely has entropy above 7.0 in `.text`.

### 2. Import table

After unpacking, the import table becomes readable. Imports tell you what the binary *can* do before you run it:

| Import | Capability |
|--------|-----------|
| `CryptGenKey`, `CryptEncrypt` | File encryption via Windows CryptoAPI |
| `RegCreateKeyExW`, `RegSetValueExW` | Registry persistence |
| `CreateProcessW`, `ShellExecuteW` | Process spawning |
| `InternetOpenW`, `InternetConnectW` | Network / C2 |
| `FindFirstFileW`, `FindNextFileW` | Filesystem enumeration |
| `DeleteFileW` | File deletion (shadow copies, originals) |

Crypto + registry + filesystem enumeration together = strong ransomware indicators.

> 📷 *[Placeholder — PEStudio import table]*

### 3. String extraction with FLOSS

Plain `strings` misses stack strings and encoded blobs. FLOSS resolves these:

```
.cerber                              ← extension appended to encrypted files
DECRYPT MY FILES                     ← ransom note base name
hxxp://[redacted].onion/[victim_id] ← C2 (Tor hidden service)
vssadmin Delete Shadows /All /Quiet  ← shadow copy deletion
bcdedit /set {default} recoveryenabled No
```

The `.onion` C2 address confirms Tor-based command and control. The `vssadmin` and `bcdedit` commands are standard ransomware anti-recovery measures.

### 4. Encryption scheme (Ghidra)

Cerber uses **hybrid encryption**:

```
1. On execution   → Generate AES-256 session key (CryptGenKey)
2. Per file       → Enumerate with FindFirstFile/FindNextFile
                  → Read file content
                  → Encrypt with AES-256
                  → Write encrypted content
                  → Rename: file.docx → file.docx.cerber
3. Key escrow     → Encrypt AES key with attacker's RSA-2048 public key
                  → Send encrypted key to C2
4. Original files → Deleted
```

Without the RSA private key from the C2 server, decryption is not feasible. The AES key is gone.

> 📷 *[Placeholder — Ghidra decompiler view, encryption loop]*

---

## Dynamic Analysis — Cuckoo Report

### File system

```
Created:  %APPDATA%\{random_GUID}\cerber.exe    ← persistence copy
Created:  Desktop\DECRYPT MY FILES.txt
Created:  Desktop\DECRYPT MY FILES.html
Created:  Desktop\DECRYPT MY FILES.url
Modified: All files in user directories          ← encryption
Renamed:  *.docx → *.docx.cerber
```

### Registry

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
  "{random_name}" = "%APPDATA%\{GUID}\cerber.exe"
```

Runs on every login until the ransom is paid or the system is wiped.

### Network

```
DNS query : [random subdomain].top
UDP traffic on port 6893
```

One of Cerber's signatures: it uses **UDP on port 6893** for C2, not HTTP. This bypasses simple HTTP-based network blocks and makes traffic analysis harder.

### Processes spawned

```
cmd.exe /c vssadmin Delete Shadows /All /Quiet
cmd.exe /c bcdedit /set {default} recoveryenabled No
cmd.exe /c bcdedit /set {default} bootstatuspolicy ignoreallfailures
```

Deletes all Volume Shadow Copies, disables recovery mode — no way to roll back.

> 📷 *[Placeholder — Cuckoo process tree]*

---

## Indicators of Compromise

| Type | Value |
|------|-------|
| File extension | `.cerber` |
| Ransom notes | `DECRYPT MY FILES.txt`, `.html`, `.url` |
| Registry key | `HKCU\...\Run` with random GUID value name |
| C2 protocol | UDP port 6893 |
| Shadow copy deletion | `vssadmin Delete Shadows /All /Quiet` |
| Persistence path | `%APPDATA%\{random_GUID}\cerber.exe` |

---

## What I took from this

**Technical:**
- How to set up an isolated analysis environment that doesn't compromise the host
- The difference between static triage (fast, safe) and dynamic analysis (more complete, more risk)
- How hybrid encryption makes ransomware decryption infeasible without the private key
- Cuckoo limitations: samples that check uptime, screen resolution, or mouse movement may detect the sandbox and stay dormant

**Methodology:**
- Static first, always. Understand what the binary *can* do before you run it
- FLOSS extracts significantly more strings than `strings`
- Snapshot before every run — never analyze on a machine you care about
- Network captures (even fake) give you C2 patterns before you touch a real sample

---

## Resources

- [Mandiant FLARE-VM](https://github.com/mandiant/flare-vm)
- [FLOSS — obfuscated string solver](https://github.com/mandiant/flare-floss)
- [Cuckoo Sandbox docs](https://cuckoo.sh/docs/)
- [MalwareBazaar](https://bazaar.abuse.ch/) — sample repository
- [Any.run](https://app.any.run/) — interactive online sandbox
- Kaspersky, Cisco Talos, Trend Micro reports on Cerber (2016–2017)