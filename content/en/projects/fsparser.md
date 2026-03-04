---
title: "FSParser"
date: 2023-06-01
draft: false
tags: ["forensics", "filesystems", "python", "low-level"]
description: "A FAT32 & EXT filesystem parser written in Python."
github: "https://github.com/Ily455/FSParser"
---

A filesystem parser for FAT32 and EXT filesystems written in Python.

## What it does

Parses raw disk images and extracts filesystem structures — directory entries, inodes, file allocation tables — without relying on the OS to mount the filesystem.

## Why I built it

Understanding how filesystems work at the raw byte level is useful for both forensic analysis and reverse engineering stored data. Built this to go deeper than what tools like Autopsy expose to the analyst.

## What I learned

- FAT32 and EXT filesystem internals — structures, offsets, data layout
- How forensic tools operate at the raw level
- Parsing binary formats in Python
