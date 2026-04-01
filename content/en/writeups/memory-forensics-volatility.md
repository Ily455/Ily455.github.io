---
title: "Memory Forensics with Volatility — WinXP SP2 Dump Analysis"
date: 2024-06-01
draft: false
description: "Full walkthrough of a Windows XP SP2 memory dump using Volatility 2.6.1 — 14 plugins covering process enumeration, hidden process detection, network connections, registry artifacts, kernel modules, and live memory inspection via volshell."
tags: ["forensics", "volatility", "memory-forensics", "windows", "dfir", "blue-team"]
---

> Lab from my SICS M2 coursework (2024). Objective: perform both basic and advanced memory forensics on a WinXP SP2 dump — from profile identification to interactive memory inspection with volshell. All commands are Volatility 2 syntax.

---

## Target

```
File    : dump_practice.dmp
Profile : WinXPSP2x86
Capture : 2016-01-03 23:00:28 UTC
Image type (Service Pack) : 2
Number of Processors : 1
KDBG : 0x8054c060L
```

The dump was captured 2016-01-03 but most processes started 2015-12-23 — the machine had been running for ~11 days before the capture.

---

## 1. Profile Identification — `imageinfo`

```bash
volatility -f /dumps/dump_practice.dmp imageinfo
```

```
Suggested Profile(s) : WinXPSP2x86, WinXPSP3x86 (Instantiated with WinXPSP2x86)
AS Layer1 : IA32PagedMemory (Kernel AS)
AS Layer2 : VirtualBoxCoreDumpElf64 (Unnamed AS)
AS Layer3 : FileAddressSpace (/dumps/dump_practice.dmp)
PAE type  : No PAE
DTB       : 0x39000L
KDBG      : 0x8054c060L
Image date and time : 2016-01-03 23:00:28 UTC+0000
```

`AS Layer2` shows `VirtualBoxCoreDumpElf64` — the dump was taken from a VirtualBox guest. `imageinfo` narrows the profile to `WinXPSP2x86`, which we use for all subsequent commands.

---

## 2. Process Listing — `pslist`

```bash
volatility -f /dumps/dump_practice.dmp pslist
```

`pslist` walks the kernel's `PsActiveProcessHead` doubly-linked list — the official OS process list. Selected output:

```
Offset(V)  Name             PID   PPID  Thds  Hnds  Start
0x823c89c8 System             4      0    51   249   —
0x821b4020 smss.exe         492      4     4     3   2015-12-23 09:25:39
0x8229f020 csrss.exe        560    492    13   390   2015-12-23 09:25:39
0x8217b510 winlogon.exe     584    492    16   418   2015-12-23 09:25:39
0x82176b28 services.exe     628    584    16   263   2015-12-23 09:25:39
0x821b7da0 lsass.exe        640    584    20   344   2015-12-23 09:25:39
0x820fdda0 explorer.exe    1596   1568    12   379   2015-12-23 00:25:42
0x8230ada0 mspaint.exe     1772   1596     4    98   2015-12-23 01:39:56
0x820095c8 wmplayer.exe    1360   1776    29   701   2015-12-23 00:31:30
0x82077020 notepad.exe     1260   1596     0   ——    2015-12-23 01:20:49 (exited 01:40:40)
0x81f819c8 notepad.exe     1788   1596     0   ——    2015-12-23 01:42:15 (exited 2016-01-03 22:55:04)
0x81fc8020 notepad.exe     1088   1596     1    27   2016-01-03 22:56:02
```

Notable: 4 notepad.exe instances — 3 already exited, 1 (PID 1088) still live at capture time. `wmplayer.exe` PID 1360 has an unusual parent (PID 1776, which is `wpabaln.exe`).

---

## 3. Hidden Process Detection — `psscan`

```bash
volatility -f /dumps/dump_practice.dmp psscan
```

`psscan` scans raw physical memory for `_EPROCESS` structures by signature — not through the OS-maintained linked list. This can reveal processes hidden by rootkits that unlink themselves from `PsActiveProcessHead`.

Result: in this dump, `psscan` output matches `pslist` exactly. No hidden processes detected.

This is the key forensic check: if `psscan` shows a process not in `pslist`, it was actively hidden from the OS list — a strong rootkit indicator.

---

## 4. Process Handles — `handles`

```bash
volatility -f /dumps/dump_practice.dmp handles -p 1772
```

Lists all kernel handles open by `mspaint.exe` (PID 1772). Handles are references to OS resources — files, registry keys, events, mutexes, etc.:

```
Offset(V)   Pid  Handle  Type             Details
0xe10096a0 1772  0x4     KeyedEvent       CritSecOutOfMemoryEvent
0x81fffbb0 1772  0xc     File             \Device\HarddiskVolume1\Documents and Settings\pero
0x8214b810 1772  0x1c    File             \Device\HarddiskVolume1\WINDOWS\WinSxS\x86_Microsoft.Windows.Common-Controls_...
0x8215b810 1772  0x28    WindowStation    WinSta0
0x82179038 1772  0x2c    Desktop          Default
0xe1129550 1772  0x34    Key              MACHINE
0x82166f38 1772  0x38    Semaphore        shell.{A48F1A32-A340-11D1-BC6B-00A0C90312E1}
0x81fd8ca0 1772  0x40    File             \Device\KsecDD
0xe193e0f0 1772  0x50    Key              MACHINE\SOFTWARE\MICROSOFT\WINDOWS NT\CURRENTVERSION\DRIVERS32
```

The open file handle on `\Documents and Settings\pero` shows the active user profile. `\Device\KsecDD` is the kernel security support provider — present in most GUI processes for credential operations.

---

## 5. Registry Execution Artifacts — `userassist`

```bash
volatility -f /dumps/dump_practice.dmp userassist
```

`userassist` extracts Windows' UserAssist registry keys — ROT13-encoded execution history stored in `NTUSER.DAT`. These keys record what programs the user launched, how many times, and when.

```
Registry : \Device\HarddiskVolume1\Documents and Settings\pero\NTUSER.DAT
Path     : Software\Microsoft\Windows\CurrentVersion\Explorer\UserAssist\{75048700-EF1F-11D0-9888-006097DEACF9}\Count

UEME_RUNPIDL:%csidl2%\MSN.lnk
  Count        : 14
  Last updated : 2015-12-23 00:22:13 UTC+0000

UEME_RUNPIDL:%csidl2%\Windows Media Player.lnk
  Count        : 14
  Last updated : 2015-12-23 00:31:20 UTC+0000

UEME_RUNPIDL:%csidl2%\Accessories\Notepad.lnk
  Count        : 1
  Last updated : 2016-01-03 22:56:02 UTC+0000
```

User `pero` launched MSN and Windows Media Player 14 times each in the first session (2015-12-23). Notepad was launched once on 2016-01-03 — matching the live `notepad.exe` PID 1088 we see in pslist.

The GUID `{75048700-EF1F-11D0-9888-006097DEACF9}` corresponds to the `UEME_RUNPIDL` class (items launched from the Start Menu Programs folder via shortcut links).

---

## 6. Kernel Modules — `modules`

```bash
volatility -f /dumps/dump_practice.dmp modules
```

`modules` walks the `PsLoadedModuleList` to enumerate kernel-mode modules (drivers, the kernel itself). These run with full ring-0 privileges:

```
Offset(V)   Name           Base        Size    File
0x823fc3a0  ntoskrnl.exe   0x804d7000  0x214200  \WINDOWS\system32\ntoskrnl.exe
0x823fc338  hal.dll        0x806ec000  0x13d80   \WINDOWS\system32\hal.dll
0x823fc2d0  kdcom.dll      0xf8a50000  0x2000    \WINDOWS\system32\KDCOM.DLL
0x823fc1f8  ACPI.sys       0xf8501000  0x2e000   ACPI.sys
0x823fc188  WMILIB.SYS     0xf8a52000  0x2000    \WINDOWS\system32\DRIVERS\WMILIB.SYS
0x823fc120  pci.sys        0xf84f0000  0x11000   pci.sys
0x823eddd8  disk.sys       0xf8580000  0x9000    disk.sys
```

Rootkits commonly load as kernel modules or hide existing modules from this list. Cross-referencing `modules` against `driverscan` (below) catches modules hidden by manipulating `PsLoadedModuleList`.

---

## 7. Driver Scan — `driverscan`

```bash
volatility -f /dumps/dump_practice.dmp driverscan
```

`driverscan` scans physical memory for `_DRIVER_OBJECT` structures by pool tag — independent of the module linked list. Differences between `driverscan` and `modules` output indicate hidden drivers.

Notable driver found: `PROCEXP141` — Process Explorer's kernel driver, which matches `procexp.exe` (PID 1204) in the process list. This confirms Process Explorer was running with its kernel driver loaded.

```
0x0000000021756c0  3  0 0xf8908000  0x4380 PROCEXP141  PROCEXP141  \Driver\PROCEXP141
```

---

## 8. Network Connections — `connections`

```bash
volatility -f /dumps/dump_practice.dmp connections
```

```
Offset(V)   Local Address      Remote Address    Pid
0x81feed00  10.0.2.15:1242     50.31.192.83:554  1360
```

One active TCP connection: local port 1242 → remote port **554** (RTSP — Real Time Streaming Protocol). PID 1360 is `wmplayer.exe`. Windows Media Player was streaming media from `50.31.192.83` over RTSP — consistent with the high handle count (701) and the 14 WMP UserAssist entries.

RTSP port 554 is the standard port for streaming media servers; this is legitimate behavior from WMP, but in a suspicious binary it would be a significant C2 indicator.

---

## 9. IE History — `iehistory`

```bash
volatility -f /dumps/dump_practice.dmp iehistory
```

```
Process: 1596 explorer.exe
Cache type "DEST" at 0x15ceef
URL: pero@http://sc1.slable.com:8126
Last accessed: 2015-12-23 01:23:20 UTC+0000

Process: 1596 explorer.exe
Location: Visited: pero@about:Home
Location: Visited: pero@res://C:\WINDOWS\system32\shdoclc.dll\dnserror.htm
```

User `pero` had IE open with a custom URL on `sc1.slable.com:8126` and visited the IE default error page (`dnserror.htm`) — which typically appears when DNS fails or the network is unavailable. The `about:Home` visit is the IE start page.

---

## 10. Console History — `consoles`

```bash
volatility -f /dumps/dump_practice.dmp consoles
```

```
ConsoleProcess: csrss.exe Pid: 560
Console: 0x4e23b0 CommandHistorySize: 50
HistoryBufferCount: 1 HistoryBufferMax: 4
Title: ??\WINDOWS\system32\cmd.exe
```

A `cmd.exe` session was open (managed by `csrss.exe`). The command buffer size is 50 but only 1 history buffer active, suggesting the console was open but few commands were typed. Actual command content was not recovered in this dump.

---

## 11. Thread Inspection — `threads`

```bash
volatility -f /dumps/dump_practice.dmp threads -p 1088
```

`threads` enumerates `_ETHREAD` kernel structures for PID 1088 (`notepad.exe`):

```
ETHREAD: 0x81f99da8  Pid: 1088  Tid: 1488
Created: 2016-01-03 22:56:02 UTC+0000
Owning Process: notepad.exe
State: Waiting:WrUserRequest
BasePriority: 0x8
StartAddress: 0x7c810867 kernel32.dll
```

State `WrUserRequest` means the thread is blocked waiting for user input — exactly what you'd expect from a Notepad window open and idle. Start address in `kernel32.dll` is the standard thread creation point for Win32 applications.

---

## 12. Interactive Memory Inspection — `volshell`

```bash
volatility -f /dumps/dump_practice.dmp volshell -p 1088
```

`volshell` provides an interactive Python shell with full access to the process's address space. Current context: `notepad.exe @ 0x81fc8020, pid=1088`.

**Disassembly at EIP:**

```python
>>> dis(0x7c90eb94)
0x7c90eb94 c3          RET
0x7c90eb95 8da42400000000  LEA ESP, [ESP+0x0]
0x7c90eb9c 8d642400        LEA ESP, [ESP+0x0]
...
0x7c90ebac 55          PUSH EBP
0x7c90ebad 8bec        MOV EBP, ESP
0x7c90ebaf 9c          PUSHF
0x7c90ebb0 81ecd0020000    SUB ESP, 0x2d0
...
0x7c90eba9 cd2e        INT 0x2e
0x7c90ebab c3          RET
```

This is `kernel32.dll` territory — the disassembly shows a `RET`, NOP sled, then a full function prologue. The `INT 0x2e` instruction is Windows XP's legacy system call gate (later replaced by `SYSENTER`).

**Memory dump at ESP:**

```python
>>> dd(0x0007febc)
0007febc  77d4919b 77d491ce 0007fefc 00000000
0007fecc  00000000 00000000 00000000 0007ff1c
0007fedc  01002a1b 0007fefc 00000000 00000000
0007feec  00000000 00000000 7c80b529 000a2332
```

Stack data at ESP shows return addresses in `ntdll.dll` range (`0x77xxxxxx`) and `kernel32.dll` (`0x7c8xxxxx`).

---

## 13. Text Extraction from Notepad — `editbox`

```bash
volatility -f /dumps/dump_practice.dmp editbox -p 1088
```

`editbox` reads the Win32 edit control buffer directly from the process's GUI state:

```
Wnd Context    : 0\WinSta0\Default
Process ID     : 1088
ImageFileName  : notepad.exe
nChars         : 17
undoBuf        : Mhm.

You are the best.
```

Then using the `notepad` plugin for all instances:

```bash
volatility -f /dumps/dump_practice.dmp notepad
Process: 1088
Text:
You are the best. Mhm. g
```

The live notepad window (PID 1088) contained the text **"You are the best. Mhm. g"**. The undo buffer held "Mhm." — the last edit operation was appending " Mhm." to the text. This is direct extraction from process memory without opening the file.

---

## 14. Virtual Address Descriptors — `vadinfo`

```bash
volatility -f /dumps/dump_practice.dmp vadinfo -p 1088
```

VADs describe how virtual memory is laid out for a process — which regions are private, mapped files, or image sections:

```
VAD node @ 0x82167318  Start 0x00020000  End 0x00020fff  Tag VadS
  Flags: CommitCharge: 1, MemCommit: 1, PrivateMemory: 1, Protection: 4
  Protection: PAGE_READWRITE

VAD node @ 0x8200f7a8  Start 0x01000000  End 0x01013fff  Tag VadI
  Flags: CommitCharge: 3, ImageMap: 1, Protection: 7
  Protection: PAGE_EXECUTE_WRITECOPY
  FileObject @ 820759e0, Name: \Device\HarddiskVolume1\WINDOWS\system32\notepad.exe
```

`VadS` (private stack/heap regions) have `PAGE_READWRITE`. The `VadI` entry at `0x01000000` is the mapped image of `notepad.exe` itself — `PAGE_EXECUTE_WRITECOPY` is the standard protection for executable image sections in Windows.

VAD inspection is useful for detecting process injection: a `PAGE_EXECUTE_READWRITE` region that doesn't correspond to a mapped image is a strong shellcode indicator.

---

## Key Takeaways

| Finding | Plugin | Significance |
|---------|--------|-------------|
| Profile: WinXPSP2x86 | `imageinfo` | Foundation for all other commands |
| No hidden processes | `psscan` vs `pslist` | No rootkit active in this dump |
| WMP → RTSP stream to 50.31.192.83:554 | `connections` | PID 1360, legitimate streaming |
| User "pero", 14× WMP, 1× Notepad | `userassist` | Execution timeline reconstruction |
| PROCEXP141 driver loaded | `driverscan` | Process Explorer was running with kernel driver |
| Notepad text: "You are the best. Mhm." | `editbox` / `notepad` | Live process memory extraction without file access |
| notepad.exe VAD mapped at 0x01000000 | `vadinfo` | Normal image mapping, no injection detected |

The pslist/psscan comparison is the core hidden process detection technique. The editbox extraction demonstrates that memory forensics can recover document content even if the file was never saved to disk.

---

## Resources

- [Volatility 2 Command Reference](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference)
- [Windows Process Internals — Memory Forensics (imphash)](https://imphash.medium.com/windows-process-internals-a-few-concepts-to-know-before-jumping-on-memory-forensics-part-4-16c47b89e826)
- [The Art of Memory Forensics — Ligh, Case, Levy, Walters](https://www.memoryanalysis.net/)
- [Volatility Foundation](https://volatilityfoundation.org/)
