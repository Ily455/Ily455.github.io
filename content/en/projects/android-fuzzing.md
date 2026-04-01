---
title: "Android Fuzzing Study"
date: 2023-05-01
draft: false
tags: ["fuzzing", "android", "droid-ff", "radamsa", "adb", "security"]
description: "Automated fuzzing of Android .dex files using Droid-FF and Radamsa on a Genymotion emulator. Confirmed crashes in libz.so, analyzed tombstones, traced the fault to adler32+227."
---

> Academic project — spring 2023. Goal: learn and practice automated fuzzing on Android. Approach: use Droid-FF to generate mutated .dex files, push them to an emulated device, run dexdump, and trace real crashes through the full triage pipeline.

---

## Setup

**Host**: Kali Linux
**Emulator**: Genymotion (Samsung Galaxy S4 image — 192.168.1.114:5555)
**Fuzzing framework**: [Droid-FF](https://github.com/antojoseph/droid-ff)
**Mutation engine**: Radamsa
**Connection**: ADB over TCP

The emulator was registered with Droid-FF via `adb connect 192.168.1.114:5555`. All file transfer and crash collection goes through ADB.

---

## Workflow

Droid-FF exposes a five-step menu:

```
(0) Generate Files         — mutate a seed file into N samples
(1) Start running fuzzer   — push samples, run target, log crashes
(2) View Crashes           — pull logcat, identify crashing inputs
(3) Triage Crashes         — confirm reproduction, collect tombstones
(4) View Source of Crashes — symbolize crash addresses
(5) Exploitability Test
```

### Step 0 — Generate files

Selected Radamsa as the mutation engine with a seed `.dex` file. Generated **40 samples**.

### Step 1 — Run the fuzzer

For each sample, Droid-FF:
1. Pushes the `.dex` to `/data/local/tmp` via `adb push`
2. Runs `adb shell /system/xbin/dexdump /data/local/tmp/sampleN.dex`
3. Logs SIGSEGV crashes from logcat

### Step 2 — View crashes

Pulled logcat output: **1375 lines** total. Multiple `.dex` files triggered SIGSEGV in dexdump.

### Step 3 — Triage crashes

For each crashing input, Droid-FF:
1. Clears `/data/tombstones/*`
2. Re-pushes the file and re-runs dexdump
3. Confirms the crash reproduces
4. Pulls the tombstone to `fuzzer/confirmed_crashes/`

Example — `sample27.dex`:
```
adb push .../sample27.dex /data/local/tmp        → 0.9 MB/s, 553109 bytes
adb shell /system/xbin/dexdump /data/local/tmp/sample27.dex
→ tombstone_00 created (10447 bytes, 2023-01-11 20:57)
adb pull /data/tombstones/tombstone_00 .../confirmed_crashes/tombstone_sample27.dex
→ 7.9 MB/s, 10447 bytes
```

### Step 4 — View source of crashes

Option 4 itself errored. Used `dmesg` to read the tombstone buffer:

```
signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0xf7609000
pid: 2600, tid: 2600, name: dexdump  >>> /system/xbin/dexdump <<<
ABI: 'x86'

eax f7609000  ebx 565a4f10  ecx 0002ff5f  edx 00000000
esi 00000023  edi 00000000

backtrace:
  #00 pc 000018b3  /system/lib/libz.so (adler32+227)
  #01 pc 0000d4db  /system/xbin/dexdump
  #02 pc 0000f0ff  /system/xbin/dexdump
  #03 pc 00006922  /system/xbin/dexdump
  #04 pc 00001b22  /system/xbin/dexdump
  #05 pc 00012a64  /system/lib/libc.so (__libc_init+100)
```

Pulled `libz.so` from the device and ran addr2line:

```bash
adb pull /system/lib/libz.so
addr2line -f -e libz.so 000018b3
# → adler32
# → ??:?
```

The function is `adler32` in `libz.so` at offset `0x18b3`. Source line info is unavailable (binary stripped, no debug symbols). The crash path: malformed `.dex` → dexdump decompression → libz adler32 → SIGSEGV at `0xf7609000`.

---

## Limitations

**Emulator vs real hardware**: Genymotion uses x86 hardware virtualization. Real Android devices run ARM. The fault address and register state differ between platforms; a crash on the emulator doesn't guarantee the same crash on hardware, and vice versa.

**No debug symbols**: `addr2line` identified the function name but returned `??:?` for source line — the device's libz.so is stripped. To go deeper, you'd need a debug build of the AOSP.

**Option 4 failure**: Droid-FF's built-in source view errored out, requiring a manual dmesg fallback. Not blocking, but a gap in the tool's workflow.

---

## What I learned

- Full mutation-based fuzzing loop: generate → push → run → crash → triage → symbolize
- Reading Android tombstones and mapping crash addresses to symbols with addr2line
- How Droid-FF coordinates ADB, crash detection, and tombstone collection
- Where emulation falls short for hardware-specific crash analysis

---

## Resources

- [Droid-FF](https://github.com/antojoseph/droid-ff)
- [Radamsa](https://gitlab.com/akihe/radamsa)
- [Android tombstone format](https://source.android.com/docs/core/tests/debug/native-crash)
- [addr2line documentation](https://man7.org/linux/man-pages/man1/addr2line.1.html)
