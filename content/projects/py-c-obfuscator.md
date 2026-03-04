---
title: "Py-C-obfuscator"
date: 2024-06-01
github: "https://github.com/Ily455/Py-C-obfuscator"
---

A source-to-source C obfuscator written in Python. Takes C source code as input and outputs a functionally equivalent but obfuscated version.

### What it does

Applies a set of obfuscation transformations at the source code level before compilation — making the code harder to read and analyze without affecting its behavior.

### Why I built it

As part of my research on obfuscation at Secure-IC, I wanted to explore obfuscation at the source level as a complement to compiler-level approaches. Python was a natural choice for rapid prototyping of AST transformations.

### What I learned

- Source-level vs IR-level obfuscation tradeoffs
- C AST parsing and manipulation in Python
- How much (or how little) source-level obfuscation survives compiler optimizations
