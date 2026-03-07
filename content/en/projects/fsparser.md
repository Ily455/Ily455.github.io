---
title: "FSParser"
date: 2026-03-06
draft: false
description: "A Python tool that parses raw disk images and extracts filesystem metadata directly from binary structures — MBR partition tables, FAT32 BPB, and EXT2/3/4 superblocks — with no OS calls."
tags: ["python", "filesystems", "forensics", "low-level", "mbr", "fat32", "ext4"]
---

## What It Does

FSParser takes a raw disk image file and extracts filesystem structures directly from binary data — no OS calls, no filesystem drivers, no mounting.

```bash
python FSParser.py disk.img
```

It reads fixed offsets, parses binary fields, and prints human-readable output about the partition layout and filesystem metadata.

---

## Why Parse Raw Structures

Most tools that "read" a filesystem rely on the OS to mount it first. That works fine in normal usage, but in forensics and low-level analysis contexts, you often need to:

- Examine a disk from a compromised system without mounting it (mounting can alter timestamps, trigger journal recovery)
- Analyze filesystems the host OS can't mount (corrupted superblock, unknown filesystem type)
- Understand exactly what the binary structures look like — not what the OS tells you

FSParser demonstrates that there's no magic: a filesystem is just carefully placed binary data at known offsets. If you know the spec, you can read it with `struct.unpack`.

---

## What It Parses

### MBR (Master Boot Record)

The MBR lives at the very first 512 bytes of a disk. FSParser reads the **partition table** at offset 446 — four 16-byte entries describing the partitions on the disk.

Each partition entry gives:
- Boot flag (is this the bootable partition?)
- Partition type (FAT32, Linux, Extended, etc.)
- Start LBA (which sector the partition begins at)
- Size in sectors

```
Partition 1: type=0x0B (FAT32 CHS), start_lba=2048, size=2097152
Partition 2: type=0x83 (Linux), start_lba=2099200, size=20971520
```

### FAT32 — BIOS Parameter Block (BPB)

The BPB is the header of a FAT32 filesystem. It starts at offset 0 of the partition (i.e., at `start_lba * 512` from the disk start) and contains all parameters needed to navigate the filesystem.

Fields extracted:

| Field | Description |
|-------|-------------|
| Bytes per sector | Almost always 512 |
| Sectors per cluster | Cluster size = this × bytes per sector |
| Reserved sectors | Sectors before the first FAT |
| Number of FATs | Usually 2 (redundancy) |
| Total sectors | Total size of the volume |
| Sectors per FAT | Size of one FAT table |
| Root cluster | Cluster number of the root directory |
| Volume label | 11-byte string |
| OEM Name | 8-byte string ("MSDOS5.0" etc.) |

### EXT2/3/4 — Superblock

The EXT superblock starts at byte offset **1024** from the start of the partition (the first 1024 bytes are reserved for a potential boot sector). It's 1024 bytes long and contains all metadata about the filesystem.

Fields extracted:

| Field | Description |
|-------|-------------|
| Inode count | Total number of inodes |
| Block count | Total number of blocks |
| Block size | `1024 << s_log_block_size` |
| Blocks per group | Used to compute block group layout |
| Inodes per group | Used to compute inode table locations |
| Magic number | `0xEF53` — confirms this is an EXT filesystem |
| Filesystem state | Clean / Errors |
| OS type | Linux, Hurd, FreeBSD, etc. |
| Volume name | 16-byte label |
| Last mount path | 64-byte string of last mount point |
| Feature flags | `has_journal` (ext3), `extents` (ext4), etc. |

The magic number `0xEF53` check is the first thing FSParser does on a potential EXT partition — if it's not there, it's not EXT.

---

## Implementation — Reading Binary Structures

All parsing is done with Python's `struct` module. The key is knowing the exact offset and format of each field.

```python
import struct

# Read EXT superblock
with open("disk.img", "rb") as f:
    f.seek(partition_offset + 1024)  # superblock at +1024
    data = f.read(1024)

# Parse magic number (little-endian unsigned short at offset 56)
magic = struct.unpack_from("<H", data, 56)[0]
if magic != 0xEF53:
    raise ValueError("Not an EXT filesystem")

# Parse inode count (little-endian unsigned int at offset 0)
inode_count = struct.unpack_from("<I", data, 0)[0]

# Parse block size (compute from log value at offset 24)
log_block_size = struct.unpack_from("<I", data, 24)[0]
block_size = 1024 << log_block_size
```

All fields are little-endian — both FAT32 and EXT use little-endian byte order, which is `<` in Python's struct format strings.

---

## Output Example

```
=== MBR Partition Table ===
Partition 1: type=0x0B (FAT32), start=2048, size=2097152 sectors
Partition 2: type=0x83 (Linux), start=2099200, size=20971520 sectors

=== FAT32 BPB (Partition 1) ===
Volume label    : MYUSB
OEM Name        : MSDOS5.0
Bytes/sector    : 512
Sectors/cluster : 8
Cluster size    : 4096 bytes
Reserved sectors: 32
FAT copies      : 2
Total sectors   : 2097152
Volume size     : ~1.0 GB

=== EXT Superblock (Partition 2) ===
Magic           : 0xEF53 ✓
Volume name     : linux-root
Block size      : 4096 bytes
Inode count     : 1310720
Block count     : 2621440
FS state        : Clean
Features        : has_journal, extents, 64bit, flex_bg
Last mounted at : /
```

---

## Getting Test Images

You don't need a real disk — you can create test images on Linux in seconds:

```bash
# Create a FAT32 image
dd if=/dev/zero of=fat32.img bs=1M count=64
mkfs.fat -F 32 fat32.img

# Create an EXT4 image
dd if=/dev/zero of=ext4.img bs=1M count=128
mkfs.ext4 ext4.img
```

Pass either image to FSParser and you'll get a full breakdown of the structures.

---

## What I Learned

- How MBR partition tables are laid out and why 512-byte sector alignment matters
- The difference between FAT12, FAT16, and FAT32 — and why the BPB structure differs between them
- Why EXT magic numbers exist and how the kernel uses them at mount time
- That `struct.unpack` is all you need to parse binary formats if you have the spec
- The distinction between EXT2, EXT3, and EXT4 at the superblock level (journal feature bit, extent tree support, 64-bit block addressing)

---

## Supported Filesystems

| Filesystem | Structures parsed | Notes |
|-----------|-------------------|-------|
| FAT32 | BPB, partition entry | Full metadata extraction |
| EXT2/3/4 | Superblock | Feature detection, size info |
| MBR | Partition table (all 4 entries) | Type identification |
| GPT | — | Not yet supported |
| NTFS | — | Not yet supported |

---

## Resources

- [FAT32 File System Specification (Microsoft)](https://download.microsoft.com/download/1/6/1/161ba512-40e2-4cc9-843a-923143f3456c/fatgen103.doc)
- [The Second Extended Filesystem — kernel.org](https://www.kernel.org/doc/html/latest/filesystems/ext2.html)
- [EXT4 documentation](https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html)
- [Python struct module](https://docs.python.org/3/library/struct.html)
- [OSDev wiki — MBR](https://wiki.osdev.org/MBR_(x86))