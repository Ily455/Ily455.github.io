---
title: "Android Fuzzing — Methodology and Lessons Learned"
date: 2023-05-01
draft: false
description: "A deeper look at the methodology behind the Android Droid-FF fuzzing project: triage pipeline, tombstone analysis, and what the libz.so crash actually tells you."
tags: ["fuzzing", "android", "droid-ff", "radamsa", "adb", "security-research"]
---

> This writeup goes deeper on methodology and analysis choices from the Android fuzzing project. For the setup and results summary, see the [project page](/projects/android-fuzzing/).

---

## Why .dex files

The target for this fuzzing campaign was `.dex` files processed by `dexdump` — not multimedia files or the media stack. The reasoning: `dexdump` is a system binary that parses a structured binary format (Dalvik Executable), processes it into memory, and uses system libraries (including `libz.so` for decompression). It's a standalone target that can be fuzzed directly over ADB without needing to trigger user-facing app behavior — no intent system, no sandbox, no HAL.

This is different from approaches that target `mediaserver` or `stagefright`. Those require triggering media parsing through Android's IPC layer, which adds several failure points. `dexdump` is reachable with a single `adb shell` command.

---

## The triage pipeline

The triage mechanism is the core of what makes Droid-FF useful. The general fuzzing loop is:

```
Generate → Push → Execute → Log → Identify crashes → Confirm → Symbolize
```

Each of these is worth examining:

### Identifying crashes from logcat

Logcat output is noisy. System processes crash independently of your fuzzing, and all crash signals end up in the same stream. Droid-FF handles this by logging a marker before each test input is executed — so you can correlate a SIGSEGV in the log to the input that preceded it.

In practice, 1375 lines of logcat output were generated, with crashes from multiple `.dex` samples visible. The structured marker approach makes this parseable without manual inspection of every line.

### Triage: reproduction and tombstone collection

Option 3 re-runs only the crashing inputs:

1. Clear `/data/tombstones/*`
2. Push the crashing sample
3. Execute `dexdump` on it
4. Check if a tombstone was created
5. If yes — pull the tombstone

The tombstone (`/data/tombstones/tombstone_00`) is Android's crash dump. It contains:
- Signal and fault address
- Register state at crash time
- Full backtrace with PC values per frame
- Open file descriptors and memory map

For `sample27.dex`: tombstone created at 10447 bytes, pulled to `confirmed_crashes/tombstone_sample27.dex`.

### Symbolizing the crash

Option 4 (View Source of Crashes) errored out in Droid-FF's built-in flow. The fallback was `dmesg` to read the tombstone kernel buffer:

```
signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0xf7609000
pid: 2600, name: dexdump >>> /system/xbin/dexdump <<<
ABI: 'x86'

backtrace:
  #00 pc 000018b3  /system/lib/libz.so (adler32+227)
  #01 pc 0000d4db  /system/xbin/dexdump
  ...
```

`SEGV_ACCERR` is "address not mapped but permissions issue" — the process tried to access memory at `0xf7609000` with wrong permissions (e.g., writing to a read-only page or accessing unmapped memory). This is a more specific signal than `SEGV_MAPERR` (address not mapped at all) — it suggests a potential buffer overflow or pointer corruption that hit a protected region rather than unmapped space.

The backtrace places the fault in `libz.so` at `adler32+227`. Pulling `libz.so` from the device and running `addr2line`:

```bash
adb pull /system/lib/libz.so
addr2line -f -e libz.so 000018b3
# → adler32
# → ??:?
```

The function is identified but the source line returns `??:?` — the device's `libz.so` is stripped. To get line-level resolution you'd need a debug build of AOSP matching the device image (or the `libz.so` with symbols from the SDK).

---

## What the crash path tells us

The call chain: `dexdump` → `libz.so` → `adler32+227`

`adler32` is a checksum function used by zlib during decompression. A `.dex` file that passes Radamsa mutation may have corrupted its internal structure in a way that makes `dexdump` attempt to decompress a data section with invalid parameters. The `adler32` function is in the decompression verification path — it computes a rolling checksum over the decompressed buffer to verify integrity. A mutation that corrupts the length or offset fields in the `.dex` compressed section header could lead `adler32` to operate on an out-of-bounds buffer.

This is a crash — not a confirmed exploitable vulnerability. The SIGSEGV at a single address doesn't tell you if the input controls `eip` (which would be needed for code execution). That's what Option 5 (Exploitability Test, using `gdb`/`gdbserver`) is designed for — which wasn't run in this project.

---

## What was missing

**dexRepair**: before fuzzing, Droid-FF runs `dexRepair` on each generated sample. This repairs structural fields in the `.dex` header (checksums, offsets) so the file passes initial validation and reaches the deeper parsing code. Without this step, most mutated `.dex` files would be rejected at the header check before reaching `libz.so` decompression.

**Android 5.0**: the target was Android 5.0 (Lollipop). This matters because the ART runtime (which replaced Dalvik in 5.0) changed how `.dex` files are processed — and because `libz.so` behavior can differ across Android versions.

**Cluster vs. single device**: the conclusion of the project report notes that a cluster of Android devices would have made the campaign significantly more effective. With a single Genymotion instance, throughput was limited by ADB round-trip latency and sequential test execution.

---

## What I Learned

- How Droid-FF coordinates the full fuzzing loop: generation, push, execution, crash logging, triage, symbolization
- Reading tombstones and mapping crash addresses with `addr2line` — and why stripped binaries limit how far you can go
- `SEGV_ACCERR` vs `SEGV_MAPERR` as different crash signals with different implications
- Why `dexRepair` matters: fuzzer outputs need to survive format validation before reaching the interesting parsing code
- The gap between "crash confirmed" and "exploitable" — that's the exploitability testing step

---

## Resources

- [Droid-FF](https://github.com/antojoseph/droid-ff)
- [Android tombstone format](https://source.android.com/docs/core/tests/debug/native-crash)
- [zlib adler32 source](https://github.com/madler/zlib/blob/master/adler32.c)
- [addr2line documentation](https://man7.org/linux/man-pages/man1/addr2line.1.html)
