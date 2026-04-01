---
title: "LLVM Obfuscation Study — Secure-IC Internship"
date: 2024-07-01
draft: false
description: "Comparative analysis of 4 LLVM-based obfuscators (O-LLVM, Hikari, Pluto, Tigress) benchmarked on AES code, followed by a PoC integration of Hikari into Secure-IC's Securyzr iSE firmware build chain."
tags: ["llvm", "obfuscation", "reverse-engineering", "embedded", "aes", "risc-v", "secure-ic"]
---

> End-of-studies internship at **Secure-IC** (Rennes, Feb–Jul 2024). Team: HOST SW Agent Program, Chip to Cloud Department. Supervisor: Sofiane TAKARABT.

---

## Context

Secure-IC's **Securyzr iSE** is a hardware security module based on a RISC-V processor. It handles cryptographic operations, secure boot, key provisioning, and device identity for IoT/automotive/defense markets. The firmware running on it contains sensitive IP.

The goal: evaluate obfuscation tools, select one, and integrate it into the Securyzr build system to protect sensitive firmware code from reverse engineering.

---

## Comparative Analysis

### Environment

Four Ubuntu Server 22.04.4 VMs in VirtualBox 7.0 on a Windows 10 host (Dell, i5-1135G7, 16GB RAM). Each VM had 3072MB RAM and 4 vCPUs — one per obfuscator, to avoid dependency conflicts.

| Obfuscator | Type | LLVM Version |
|------------|------|-------------|
| O-LLVM | LLVM plugin | 4.0.1 |
| Hikari | LLVM plugin | 15.0.2 |
| Pluto | LLVM plugin | 14.0.6 |
| Tigress | Source-to-source | — |

Test code: `prod_mat.c` (matrix multiplication) and `aes.c` (TinyAES implementation).

Measurement tools: `clock()` for execution time, `du` for binary size, `heaptrack` + `/bin/time` for memory, BinDiff + Ghidra (BinExport) for binary similarity.

### Transformations Tested

| Transform | Description |
|-----------|-------------|
| BCF | Bogus Control Flow — inserts opaque-predicate-guarded dummy blocks |
| FLA | Control Flow Flattening — routes all blocks through a dispatcher |
| SUB | Instruction Substitution — replaces operators with equivalent sequences |
| MBA | Mixed Boolean-Arithmetic |
| IB | Indirect Branching |
| FW | Function Wrapper |
| BBS | Basic Block Split |
| GLE | Globals Encryption |
| VRTLZ | Virtualization (Tigress only) — converts function to bytecode interpreter |
| TWO | BCF + FLA combined |

### Binary Size Impact

BCF and FLA introduce the most overhead. Tigress VRTLZ is extreme:

| Obfuscator | Transform | aes.c original | aes.c obfuscated | Ratio |
|------------|-----------|----------------|-----------------|-------|
| Tigress | VRTLZ | 18,672 B | 2,646,922 B | **61.2×** |
| Tigress | BCF | 47,960 B | 161,986 B | 3.75× |
| Pluto | TWO | 25,344 B | 89,848 B | 3.5× |
| O-LLVM | TWO | 25,376 B | 47,584 B | 1.88× |
| Hikari | SE | 25,440 B | 62,504 B | 2.46× |

SUB, FW, MBA and SE had minimal size impact across all tools.

### Execution Time

O-LLVM had the worst execution time overhead. Hikari was the most efficient:

- O-LLVM FLA on `prod_mat.c`: ~5× execution time ratio
- O-LLVM TWO on `prod_mat.c`: ~22× ratio
- Tigress TWO on `aes.c`: 0.9013s (excluded as outlier — 50× baseline)
- Tigress BCF on `prod_mat.c`: **segmentation fault** — binary crashed
- Hikari: consistently below 4× overhead across all transformations

### Similarity Scores (BinDiff)

Lower score = more divergence from original = stronger obfuscation:

| Obfuscator | Transform | Similarity |
|------------|-----------|------------|
| Pluto | BCF | **31.4%** |
| Tigress | BCF | 34.3% |
| O-LLVM | BCF | 55.4% |
| Hikari | BCF | 77.9% |
| All tools | SUB | ~47–48% |
| Hikari | IB | 49.2% |

BCF is the most impactful transformation across all tools. Pluto consistently produced the lowest similarity scores (most obfuscated), but with higher execution overhead than Hikari.

### Compiler Comparison: LLVM (Clang 14.0) vs GCC 11.4

- Clang takes slightly longer to compile at all optimization levels
- Both compilers improve execution time with higher `-O` levels
- Clang-compiled binaries use significantly less memory than GCC at `-O1` and above

### Impact Analysis

A clear pattern: lower similarity score correlates with higher resource cost (size + execution time). Transformations like SUB, FW, MBA, SE produce minimal CFG changes and barely affect performance. BCF and FLA meaningfully alter the binary structure but come at a resource cost.

**Hikari** offers the best tradeoff: meaningful obfuscation with acceptable overhead, built on the latest LLVM version, and it didn't produce the crashes and errors seen with Tigress.

---

## PoC — Integration into Securyzr iSE

### Obfuscator Selected: Hikari

Chosen for: widest transformation set among LLVM plugins, LLVM 15.0.2 base (most recent), best execution time performance, and no stability issues (unlike Tigress which crashed on multiple test cases).

### Build Server

| Spec | Value |
|------|-------|
| CPU | Intel Xeon Gold 6242R @ 3.10GHz |
| RAM | 64 GB |
| Storage | >2.5 TB |
| OS | Red Hat Enterprise Linux 8.5 |

### Building Hikari

```bash
cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS=clang \
  -DLLVM_TARGETS_TO_BUILD="X86;ARM;RISCV;AArch64" \
  ./llvm -B ./build
ninja
```

Multi-architecture build targeting X86, ARM, RISC-V, and AArch64 — matching Securyzr's RISC-V target and the x86 development environment.

### Target Code

Securyzr iSE runs on RISC-V and handles pre-boot NVM (Non-Volatile Memory) generation. The NVM contains firmware that must be encrypted before being sent to the Secure Element for authentication. The encryption uses a proprietary AES GCM implementation written in C — this is the code targeted for obfuscation.

Scope was limited to the AES subset (x86-compilable, no cross-compilation required) given the 6-month internship constraint.

### Build System Integration

The integration required three changes to the existing Makefile hierarchy:

1. **Compiler replacement**: switch from `gcc` to the Hikari-enabled clang
2. **Compiler flags**: add `-mllvm -<transformation>` flags for each obfuscation technique
3. **IR output**: configure the build to emit `.ll`/`.bc` files alongside `.o` files, enabling cross-architecture recompilation later

```makefile
CC = /path/to/hikari/bin/clang
CFLAGS_IR = -emit-llvm -c -mllvm -bcf  # example: Bogus Control Flow
```

The build produces a `build/` directory with:
- `obj/`: obfuscated object files
- `ir/`: LLVM IR files (platform-independent, reusable)
- `bin/`: final executable

### Validation

Two tests on the target FPGA hardware (ARM + RISC-V CPUs):

- **`prog` test** — launches the secure boot flow, which requires the NVM to be correctly generated and loaded. Passed: NVM was generated, transferred, and firmware launched successfully.
- **`CRYPTO_TESTS`** — exercises SE cryptographic functions. Passed: obfuscated AES code produced correct outputs.

The obfuscated code had no functional impact on the build process or the Securyzr boot sequence.

---

## Challenges

- **Toolchain compatibility**: switching compilers in a large C project with a hierarchy of makefiles required resolving linker and library compatibility issues
- **Cross-compilation**: Securyzr targets RISC-V; the development server is x86. The IR-first approach (emit `.ll`, recompile for target) was the path forward but hit build system complexity
- **Stability**: Tigress generated crashes during analysis — this ruled it out for production use regardless of its obfuscation strength
- **Measurement limitations**: `heaptrack` showed uniform 76.8KB across all transforms (likely due to static allocation), requiring `/bin/time` as a fallback for memory measurement

---

## What I Learned

- How to benchmark obfuscation tools systematically — size, execution time, memory, CFG similarity (BinDiff)
- Why BCF and FLA have significantly more impact than SUB or MBA at the binary level
- How to integrate an LLVM-based obfuscator into an existing multi-makefile C build system
- The difference between obfuscation strength and production suitability — Tigress produced the most obfuscated binaries but crashed on real code
- Cross-compilation constraints in a RISC-V embedded security context

---

## Resources

- [Hikari Obfuscator](https://github.com/HikariObfuscator/Hikari)
- [O-LLVM](https://github.com/obfuscator-llvm/obfuscator)
- [Pluto Obfuscator](https://github.com/bluesadi/Pluto)
- [Tigress](https://tigress.wtf/introduction.html)
- [BinDiff](https://www.zynamics.com/bindiff.html)
- [Collberg et al. — A Taxonomy of Obfuscating Transformations (1997)](https://researchspace.auckland.ac.nz/handle/2292/3491)
