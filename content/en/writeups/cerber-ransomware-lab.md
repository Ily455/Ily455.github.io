---
title: "Cerber Ransomware — Static & Dynamic Analysis Lab"
date: 2023-09-01
draft: false
description: "Full analysis of a 2017 Cerber sample under Flare-VM and Cuckoo Sandbox — packed and unpacked, behavioral tracing from DLL loading to file encryption, real IoCs."
tags: ["malware-analysis", "ransomware", "reverse-engineering", "flare-vm", "cuckoo"]
---

> Work done during my internship at **Techso Group** (Casablanca, July–August 2023). Mission: study malware analysis methodologies and perform a hands-on analysis of the Cerber ransomware family. Two passes: automated with Cuckoo first to orient the analysis, then manual under Flare-VM.

---

## What is Cerber

Cerber is a ransomware-as-a-service (RaaS) family, active primarily between 2016 and 2017. Its affiliate model made it massively distributed — operators could license the ransomware and keep a cut of ransom payments. The techniques it uses — dynamic API loading, hybrid encryption, C2 over non-HTTP channels — are still present in modern families. The goal here wasn't to find new things about Cerber, but to build a repeatable analysis workflow on a well-documented sample.

---

## Lab Setup

**Flare-VM** (manual dynamic analysis): Fresh Windows 10 VM, host-only adapter, no internet. Windows Defender and auto-updates disabled before install. Flare-VM script pulls tools via Chocolatey.

**Cuckoo Sandbox** (automated pre-analysis): Ubuntu host → KVM → Windows 7 guest with Cuckoo agent. Network routed through InetSim. Ran first to identify behavioral hotspots before manual analysis.

Key tools:

| Tool | Purpose |
|------|---------|
| Detect-It-Easy (DIE) / exeinfope | Packer/compiler detection |
| PEStudio | PE header, imports, entropy, strings |
| Ghidra | Disassembly and decompilation |
| x64dbg | Dynamic debugging |
| FLOSS | Obfuscated string extraction |
| unpac.me | Automated unpacking |

---

## Part I — Packed Sample

### Sample info

```
Name      : cerber.exe
Type      : PE32 (GUI) Intel 80386
Size      : 619,008 bytes (622,592 on disk)
Timestamp : Wed May 24 20:48:34 2017 UTC
MD5       : 8b6bc16fd137c09a08b02bbe1bb7d670
SHA-1     : c69a0f6c6f809c01db92ca658fcf1b643391a2b7
SHA-256   : e67834d1e8b38ec5864cfa101b140aeaba8f1900a6e269e6a94c90fcbfe56678
Compiler  : Microsoft Visual C/C++ (2008)
Linker    : Microsoft Linker 9.0 [GUI32]
```

### PE structure

Sections: `.text` 52.19% (executable), `.data` 0.74% (writable), `.rdata` 38.79%, `.rsrc` 8.11%

The `.rsrc` section contains an icon but no notable embedded resources. Headers have relocation info stripped.

### Packing detection

DIE and exeinfope report no known packer — which is misleading. Disassembling `.text` reveals very little actual code. The real behavior appears only at runtime through the API call pattern: `LdrGetProcedureAddress` is used to dynamically resolve `LoadLibraryExA`, `GetProcAddress`, `VirtualAlloc`, and `VirtualProtect`. This is the signature of a custom unpacking stub. The packed sample was unpacked using **unpac.me** to get the payload.

### Import table (packed version)

The packed version's import table is large and suspicious even before unpacking:

- File operations: `CreateFile`, `CopyFile`, `DeleteFile`, `MoveFile`, `FindFirstFile`, `FindNextFile`, `GetFileSize`
- Registry: `RegCreateKeyEx`, `RegDeleteKey`, `RegOpenKeyEx`, `RegSetValueEx`, `RegQueryValueEx`
- Process: `CreateProcess`, `CreateProcessAsUser`, `ShellExecuteEx`, `TerminateProcess`
- Memory: `VirtualAlloc`, `VirtualFree`, `VirtualQuery`
- GUI: `FindWindow`, `SendMessage`, `ShowWindow`, `MessageBox`
- Crypto: accessed via dynamic loading (not in static import table)

---

## Part II — Unpacked Payload

### Sample info

```
Name      : 53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924
Size      : 196,608 bytes
Timestamp : Thu May 18 09:35:03 2017
MD5       : ab0953388bf5b63bbc85cd10a7b73022
SHA-1     : 5fa927f92c01364182a93d7468ba4416802cb722
SHA-256   : 53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924
Compiler  : Microsoft Visual C/C++ (2010 SP1)
Linker    : Microsoft Linker 10.0 [GUI32]
```

Note the different compiler from the packed version — the packer stub and the payload were built separately.

### PE structure (unpacked)

Sections: `.text` 28.13%, `.rdata` 2.34%, `.data` 66.67% (writable), `.reloc` 2.34%

The `.rdata` section has entropy **7.271** — unusually high, indicating obfuscated or encrypted data embedded in read-only data. The `.data` section at 66.67% is abnormally large for writable data.

### Static imports (unpacked)

The static import table contains only `ntdll.dll`: `memmove`, `memset`, `memcpy`, `NtQueryVirtualMemory`. Everything else is loaded at runtime.

Strings in the binary reference libraries not present in the import table: `crypt32.dll`, `kernel32.dll`, `ntdll.dll`, `urlmon.dll`, `advapi32.dll` — all loaded dynamically via `LdrLoadDll` during execution.

---

## Dynamic Analysis

### Execution sequence (Cuckoo + Flare-VM)

**1. Dynamic library loading**

The malware calls `LdrLoadDll` to load its full dependency set at runtime: `advapi32`, `crypt32`, `IMM32`, `gdi32`, `mpr`, `netapi32`, `SAMCLI`, `NETUTILS`, `ole32`, `oleaut32`, `powrprof`, `shlwapi`. None of these appear in the static import table.

**2. Firewall manipulation**

Before doing anything visible, Cerber spawns `netsh.exe` to weaken the host's defenses:

```
netsh.exe advfirewall set allprofiles state on
netsh.exe advfirewall reset
netsh.exe advfirewall firewall add rule name="FktwwYI8qV" dir=out action=block program="C:/Program Files/Windows Defender/MpCmdRun.exe"
netsh.exe advfirewall firewall add rule name="OpnMEgcJzO" dir=out action=block program="C:/Program Files/Windows Defender/MSASCui.exe"
```

The last two rules block outbound network traffic from Windows Defender's command-line scanner and UI process — preventing real-time protection updates and threat reporting.

**3. C2 communication**

Cerber sends encrypted data via **UDP port 6893** to a range of IP addresses:
- 178.33.160.0 – 178.33.163.255
- 178.33.158.0 – 178.33.158.31
- 178.33.159.0 – 178.33.159.31

Implemented through `sendto` in `ws2_32`. The use of UDP (not HTTP/HTTPS) bypasses basic HTTP-layer monitoring. `InternetCrackUrl` is also called to parse Bitcoin blockchain API URLs for payment tracking.

**4. Filesystem enumeration and encryption**

The malware uses `FindFirstFileExW`, `GetFileAttributesW`, `NtQueryDirectoryFile` to enumerate the filesystem recursively. Default directories like `Pictures` remain untouched — selective targeting.

For each file:
```
NtReadFile        → read file content to buffer
CryptEncrypt      → encrypt content (key_handle: 0x004ca898)
NtWriteFile       → write encrypted data back to file
MoveFileWithProgressW → rename to random name + random extension
```

Example observed: `c:\Acrobat3\Reader\ACROBAT.PDF` → `c:\Acrobat3\Reader\9Fck7YJuLS.af22`

The encrypted file gets a **random name** with a **random extension** — not `.cerber`. This is a later variant with randomized extension behavior to evade signature-based detection.

**5. Ransom note delivery**

After encrypting each directory, Cerber drops two ransom artifacts:
- `_R_E_A_D___T_H_I_S___LHYR8QK_.hta` — HTML Application, launched via `mshta.exe`
- `_R_E_A_D___T_H_I_S___DRIDMPY_.txt` — plain text, opened via `notepad.exe`

The HTA file renders a browser-like ransom page with Tor hidden service payment links.

**6. Bitcoin payment tracking**

Before exiting, Cerber queries Bitcoin blockchain APIs using `InternetCrackUrl`:

```
http://api.blockcypher.com/v1/btc/main/addrs/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
http://btc.blockr.io/api/v1/address/txs/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
https://chain.so/api/v2/address/btc/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
```

This polls whether the victim has paid to BTC address `17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt`.

**7. Self-termination**

```
taskkill /f /im "cerber.exe"
```

Cerber terminates itself after completing the encryption process — a deliberate anti-analysis strategy to limit post-execution inspection time.

---

## Evasion Techniques

- **Custom packing**: the packed stub is undetected by common packer scanners, uses `LdrGetProcedureAddress` to reconstruct the import table at runtime
- **Randomized file extensions**: later Cerber variants abandoned the `.cerber` extension for random extensions + random filenames — breaks file-extension-based detection
- **Firewall rule injection**: blocks Windows Defender outbound traffic before encryption starts
- **Self-termination**: kills its own process post-encryption

---

## Indicators of Compromise

| Type | Value |
|------|-------|
| Packed SHA-256 | `e67834d1e8b38ec5864cfa101b140aeaba8f1900a6e269e6a94c90fcbfe56678` |
| Unpacked SHA-256 | `53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924` |
| File extension | Random (e.g. `.af22`) + random filename |
| Ransom note | `_R_E_A_D___T_H_I_S___[random].hta` / `.txt` |
| C2 protocol | UDP port 6893 |
| C2 IP range | 178.33.158.0–163.255 |
| Domains | `chain.so`, `bitaps.com`, `btc.blockr.io`, `api.blockcypher.com` |
| Bitcoin address | `17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt` |
| Firewall rules | Outbound block on `MpCmdRun.exe`, `MSASCui.exe` |

---

## What I Took From This

The most useful outcome wasn't the IoC list — it was the workflow. Cuckoo gives you an oriented starting point in 10 minutes; manual analysis under Flare-VM fills in the details.

The packing detection was a good lesson: exeinfope saying "not packed" doesn't mean not packed. The disassembly and the runtime API call pattern are more reliable signals than static packer scanners.

The Bitcoin payment tracking via blockchain APIs was unexpected — it means the attackers can verify payment status without direct C2 contact, reducing their operational exposure.

---

## Resources

- [Mandiant FLARE-VM](https://github.com/mandiant/flare-vm)
- [FLOSS — obfuscated string extractor](https://github.com/mandiant/flare-floss)
- [Cuckoo Sandbox docs](https://cuckoo.sh/docs/)
- [unpac.me — automated unpacker](https://www.unpac.me/)
- [MalwareBazaar](https://bazaar.abuse.ch/)
