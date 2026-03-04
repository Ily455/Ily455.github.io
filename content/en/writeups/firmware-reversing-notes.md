---
title: "Firmware Reversing Notes: UART, Memory Maps, and Dead Ends"
date: 2026-02-10
draft: false
---

This write-up summarizes a small firmware reversing exercise I did on an ARM-based IoT board. The objective was simple: identify the boot flow and locate where authentication decisions are made.

### Scope

- Target: stripped ARM firmware image extracted from vendor update package
- Goal: recover boot sequence and spot trust boundaries
- Tools: `binwalk`, `ghidra`, `strings`, `qemu-system-arm` (limited emulation)

### Workflow

1. **Static triage**
   - Used entropy and signature scans to split bootloader, kernel, and rootfs blobs.
   - Located a UART init routine by searching for known register constants from the SoC datasheet.

2. **Boot-path mapping**
   - Reconstructed a high-level call path from reset handler to image verification.
   - Identified a branch where signature verification failure falls back to a recovery path.

3. **Validation attempts**
   - Tried partial emulation to observe the fallback behavior.
   - Emulation was incomplete due to missing peripherals and MMIO assumptions.

### Key findings

- The secure-boot check exists, but error handling paths are complex and easy to misread without hardware traces.
- Logging strings can dramatically accelerate reverse engineering when symbols are absent.
- Emulation alone is rarely enough for embedded targets; combining static analysis with on-device traces is usually required.

### Takeaways

- Build a memory-map notebook early; it saves hours later.
- Mark uncertainty explicitly (confirmed behavior vs inferred behavior).
- Negative results (failed emulation, dead code paths) still improve the investigation quality.
