---
title: "FSParser"
date: 2023-06-01
github: "https://github.com/Ily455/FSParser"
---

A filesystem parser for FAT32 and EXT filesystems written in Python.

### What it does

Parses raw disk images and extracts filesystem structures — directory entries, inodes, file allocation tables — without relying on the OS to mount the filesystem.

### Why I built it

Built as part of my interest in forensics and low-level systems. Understanding how filesystems work at the raw byte level is useful for both forensic analysis and reverse engineering stored data.

### What I learned

- FAT32 and EXT filesystem internals — structures, offsets, data layout
- How forensic tools like Autopsy and FTK operate at the raw level
- Parsing binary formats in Python
