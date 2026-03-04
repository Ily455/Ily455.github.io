---
title: "FSParser"
date: 2023-06-01
draft: false
tags: ["forensics", "filesystems", "python", "low-level"]
description: "Un parseur de systèmes de fichiers FAT32 & EXT écrit en Python."
github: "https://github.com/Ily455/FSParser"
---

Un parseur de systèmes de fichiers FAT32 et EXT en Python.

## Ce que le projet fait

Analyse des images disque brutes et extrait les structures du système de fichiers — entrées de répertoire, inodes, tables d’allocation — sans montage via l’OS.

## Pourquoi je l’ai construit

Comprendre les systèmes de fichiers au niveau octet est utile en forensic et en reverse engineering de données stockées. J’ai construit ce projet pour aller plus loin que les abstractions des outils classiques.

## Ce que j’ai appris

- Les internals FAT32 et EXT : structures, offsets et organisation des données
- Comment les outils forensics opèrent au niveau brut
- Le parsing de formats binaires en Python
