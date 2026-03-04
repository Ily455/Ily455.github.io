---
title: "Android Fuzzing Study"
date: 2023-05-01
draft: false
tags: ["fuzzing", "android", "research", "security"]
description: "A 3-month study and implementation of fuzzing on Android — generational and mutational fuzzing, AFL, two-VM environment."
---

A study and practical implementation of fuzzing on Android, carried out as a school project over ~3 months.

## What I did

- Studied generational and mutational fuzzing techniques
- Evaluated several open-source fuzzers including AFL and Droid-FF
- Set up a two-VM environment (Linux + Android via ADB over TCP) to attempt reproducing a known Android multimedia vulnerability from 2014
- Generated large volumes of malformed multimedia samples and fed them to a vulnerable Android build

## What actually happened

No vulnerability was reproduced. Midway through, I watched a talk from a company doing professional Android fuzzing research — they mentioned that their setup required pools of physical devices because emulated environments introduce too many variables that mask real crashes. That put my two-VM setup in perspective.

The project didn't produce results, but it produced something more useful: a clear understanding of why this kind of research is hard, what infrastructure it actually requires, and where the real limitations are.

## What I learned

- Fuzzing fundamentals: generation-based vs mutation-based approaches
- ADB, Android internals, and emulation limitations
- The gap between academic fuzzing setups and production-grade research infrastructure
- How to present honest negative results
