---
title: "FSParser"
collection: talks
type: "Talk"
permalink: /projects/FSParser
# venue: "UC San Francisco, Department of Testing"
date: 2023-04-09
# location: "San Francisco, California"
---


## LINK: [FSParser](https://github.com/Ily455/FSParser)

This script parses MBR partitions, FAT32 filesystems, and EXT filesystems to extract useful information.

## Features
- Parses MBR partitions and displays partition table information, such as partition type and size.
- Parses FAT32 filesystems and displays information about the filesystem, such as total size, free space, and cluster size.
- Parses EXT filesystems and displays information about the filesystem, such as total size, free space, and inode count.
- Accepts command line arguments to control script behavior.
    - `-h`: Display help message.
    - `-m`: Parse MBR partition only.
    - `-i <image>`: Specify the image file to parse.
    - `-f`: Parse the filesystem only.