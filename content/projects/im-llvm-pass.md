---
title: "IM-LLVM-Pass"
date: 2024-07-01
github: "https://github.com/Ily455/IM-LLVM-Pass"
---

An LLVM compiler pass that performs identifier mangling — randomizing function names, global variable names, and structure names at compile time.

### What it does

The pass hooks into the LLVM compilation pipeline and renames identifiers using a randomization scheme. The goal is to make reverse engineering harder by eliminating meaningful symbol names from the compiled binary.

### Why I built it

This came out of my internship at Secure-IC, where I was studying code obfuscation techniques and their effect on binary analysis. After working with existing LLVM-based tools, I wanted to implement a pass myself to understand the internals of the compilation chain more deeply.

### What I learned

- How LLVM passes work and how to hook into the compilation pipeline
- The difference between obfuscation at the source level vs IR level vs binary level
- Limitations of identifier renaming as a standalone obfuscation technique
