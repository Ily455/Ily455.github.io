---
title: "IM-LLVM-Pass"
date: 2026-03-06
draft: false
description: "An LLVM module pass that mangles internal symbol names at the IR level using a seeded PRNG — making reverse engineering significantly harder without breaking program semantics."
tags: ["llvm", "compiler", "obfuscation", "reverse-engineering", "cpp", "low-level"]
---

## What It Does

IM-LLVM-Pass is a compiler pass that runs during the LLVM compilation pipeline and renames all internal (non-exported) functions and global variables to random-looking strings — before the binary is produced.

```
Source code → Clang → LLVM IR → [IM-LLVM-Pass] → mangled IR → Object code → Binary
```

The result: a binary where internal symbols like `checkLicenseKey` or `decrypt_payload` become `_Zf3a9b1c` — the program behaves identically, but reverse engineers can't use symbol names as a starting point.

---

## Why IR Level

The pass operates on **LLVM IR** (Intermediate Representation), not on source code or the final binary.

This matters because:
- **Source-level obfuscation** requires understanding the language's AST and semantics — fragile, language-specific
- **Binary-level patching** after compilation can break relocations and debug info
- **IR-level** is language-agnostic (any language that compiles to LLVM works), operates before final code generation, and can safely rename symbols while preserving cross-references

---

## How It Works

The pass is a **Module Pass** — it sees the entire program at once, not just one function.

```
For each function in the module:
  → Skip if external linkage (exported — renaming would break ABI)
  → Skip if named "main" (entry point must remain findable)
  → Generate a new name: seed PRNG with symbol name → produce random string
  → Rename the function everywhere it's referenced

Same for global variables.
```

### Name generation

The PRNG used is `std::mt19937` (Mersenne Twister), seeded with a hash of the original symbol name. This makes the renaming **deterministic** — same source, same pass, same output every time. Useful for reproducible builds and debugging the pass itself.

```cpp
std::mt19937 rng(std::hash<std::string>{}(originalName));
std::string mangledName = generateName(rng);
```

The generated names follow a pattern that looks like compiler-generated symbols, making them blend in rather than stand out as obviously obfuscated.

---

## Build & Use

### Prerequisites

- LLVM 14+ development headers
- CMake 3.13+
- Clang (to compile targets)

```bash
# Build the pass
mkdir build && cd build
cmake .. -DLLVM_DIR=/path/to/llvm/lib/cmake/llvm
make

# Compile a target with the pass loaded
clang -fpass-plugin=./build/libManglePass.so -O1 target.c -o target_mangled
```

### What changes

Before the pass — `nm target | grep T`:
```
000000000000 T main
000000000000 T checkLicenseKey
000000000000 T decrypt_payload
000000000000 T computeChecksum
```

After the pass:
```
000000000000 T main
000000000000 T _Zf3a9b1c
000000000000 T _Z8d2e4f1a
000000000000 T _Z1b7c9d3e
```

`main` is preserved. Everything else is gone.

---

## IR and Assembly Diff

The pass produces visible changes at the IR level:

**Before (test.ll excerpt):**
```llvm
define internal i32 @addNumbers(i32 %a, i32 %b) {
  %result = add i32 %a, %b
  ret i32 %result
}

define i32 @main() {
  %r = call i32 @addNumbers(i32 3, i32 4)
  ret i32 %r
}
```

**After (mangled-test.ll excerpt):**
```llvm
define internal i32 @_Zf3a9b1c(i32 %a, i32 %b) {
  %result = add i32 %a, %b
  ret i32 %result
}

define i32 @main() {
  %r = call i32 @_Zf3a9b1c(i32 3, i32 4)
  ret i32 %r
}
```

The call site in `main` is updated automatically — the pass handles all references.

> 📷 *[IR diff — see /example/IR-diff.png in the repo]*

> 📷 *[Assembly diff — see /example/assembly-diff.png in the repo]*

---

## Limitations

This is a learning project — not production obfuscation. Known limitations:

- **`main` is always preserved** — necessary for the binary to function, but it's a known entry point
- **External symbols untouched** — anything with external linkage keeps its name (by design — renaming would break the ABI)
- **Debug symbols** — if you compile with `-g`, DWARF debug info may still contain original names
- **No control flow obfuscation** — symbol renaming alone doesn't change the control flow graph, which is what most serious reverse engineers analyze
- **Standalone tool** — not integrated into a build system or CI pipeline; has to be explicitly loaded

For real obfuscation needs, tools like [OLLVM](https://github.com/obfuscator-llvm/obfuscator) or commercial solutions add control flow flattening, bogus control flow, and instruction substitution on top of renaming.

---

## What I Learned

Building this pass required understanding LLVM's pass infrastructure at a level that generic tutorials don't cover:

- The difference between Function Passes and Module Passes — and why symbol renaming requires module scope
- How LLVM tracks symbol references — renaming a function requires updating every `call` and `reference` in the IR, not just the definition
- Why `external` linkage symbols can't be renamed — they're part of the ABI contract with the linker
- How `std::mt19937` seeding works and why deterministic obfuscation matters for reproducibility

---

## Resources

- [LLVM Writing a Pass documentation](https://llvm.org/docs/WritingAnLLVMPass.html)
- [LLVM Language Reference (IR)](https://llvm.org/docs/LangRef.html)
- [LLVM Programmer's Manual](https://llvm.org/docs/ProgrammersManual.html)
- [OLLVM — Obfuscator-LLVM](https://github.com/obfuscator-llvm/obfuscator) — more complete obfuscation framework