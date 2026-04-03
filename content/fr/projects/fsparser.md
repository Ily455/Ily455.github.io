---
title: "FSParser"
date: 2026-03-06
draft: false
description: "Un outil Python qui parse des images disque brutes et extrait les métadonnées de systèmes de fichiers directement depuis les structures binaires — tables de partitions MBR, BPB FAT32, superblocs EXT2/3/4 — sans appel système."
tags: ["python", "systemes-fichiers", "forensics", "bas-niveau", "mbr", "fat32", "ext4"]
---

> Projet réalisé dans le cadre de mon diplôme d'ingénieur SICS, cours de forensics (2023). Construit comme exercice pratique sur les internals des systèmes de fichiers — parser des images disque brutes sans s'appuyer sur l'OS pour les monter ou les interpréter.

## Ce que ça fait

FSParser prend une image disque brute et extrait les structures de systèmes de fichiers directement depuis les données binaires — pas d'appel système, pas de drivers de fichiers, pas de montage.

```bash
python FSParser.py disk.img
```

Il lit des offsets fixes, parse des champs binaires et affiche une sortie lisible sur la disposition des partitions et les métadonnées du système de fichiers.

---

## Pourquoi parser les structures brutes

La plupart des outils qui "lisent" un système de fichiers s'appuient sur l'OS pour le monter d'abord. Ça fonctionne en usage normal, mais dans les contextes forensics et d'analyse bas niveau, on a souvent besoin de :

- Examiner un disque d'un système compromis sans le monter (le montage peut altérer les timestamps, déclencher une récupération de journal)
- Analyser des systèmes de fichiers que l'OS ne peut pas monter (superbloc corrompu, type de fichiers inconnu)
- Comprendre exactement à quoi ressemblent les structures binaires — pas ce que l'OS vous dit

FSParser démontre qu'il n'y a pas de magie : un système de fichiers n'est que des données binaires soigneusement placées à des offsets connus. Si on connaît la spec, on peut les lire avec `struct.unpack`.

---

## Ce qui est parsé

### MBR (Master Boot Record)

Le MBR se trouve dans les tout premiers 512 octets d'un disque. FSParser lit la **table de partitions** à l'offset 446 — quatre entrées de 16 octets décrivant les partitions du disque.

Chaque entrée de partition donne :
- Flag de boot (est-ce la partition bootable ?)
- Type de partition (FAT32, Linux, Extended, etc.)
- LBA de début (quel secteur la partition commence)
- Taille en secteurs

```
Partition 1 : type=0x0B (FAT32 CHS), start_lba=2048, size=2097152
Partition 2 : type=0x83 (Linux), start_lba=2099200, size=20971520
```

### FAT32 — BIOS Parameter Block (BPB)

Le BPB est l'en-tête d'un système de fichiers FAT32. Il commence à l'offset 0 de la partition (soit à `start_lba * 512` depuis le début du disque) et contient tous les paramètres nécessaires pour naviguer dans le système de fichiers.

Champs extraits :

| Champ | Description |
|-------|-------------|
| Octets par secteur | Presque toujours 512 |
| Secteurs par cluster | Taille de cluster = ce nombre × octets par secteur |
| Secteurs réservés | Secteurs avant le premier FAT |
| Nombre de FAT | Généralement 2 (redondance) |
| Total secteurs | Taille totale du volume |
| Secteurs par FAT | Taille d'une table FAT |
| Cluster racine | Numéro de cluster du répertoire racine |
| Label de volume | Chaîne de 11 octets |
| Nom OEM | Chaîne de 8 octets ("MSDOS5.0" etc.) |

### EXT2/3/4 — Superbloc

Le superbloc EXT commence à l'offset octet **1024** depuis le début de la partition (les 1024 premiers octets sont réservés pour un éventuel secteur de boot). Il fait 1024 octets et contient toutes les métadonnées sur le système de fichiers.

Champs extraits :

| Champ | Description |
|-------|-------------|
| Nombre d'inodes | Nombre total d'inodes |
| Nombre de blocs | Nombre total de blocs |
| Taille de bloc | `1024 << s_log_block_size` |
| Blocs par groupe | Utilisé pour calculer la disposition des groupes de blocs |
| Inodes par groupe | Utilisé pour calculer les emplacements des tables d'inodes |
| Nombre magique | `0xEF53` — confirme que c'est un système de fichiers EXT |
| État du système de fichiers | Propre / Erreurs |
| Type d'OS | Linux, Hurd, FreeBSD, etc. |
| Nom de volume | Label de 16 octets |
| Chemin du dernier montage | Chaîne de 64 octets du dernier point de montage |
| Flags de fonctionnalités | `has_journal` (ext3), `extents` (ext4), etc. |

La vérification du nombre magique `0xEF53` est la première chose que fait FSParser sur une partition EXT potentielle — si ce n'est pas là, ce n'est pas de l'EXT.

---

## Implémentation — Lecture des structures binaires

Tout le parsing est fait avec le module `struct` de Python. La clé est de connaître l'offset exact et le format de chaque champ.

```python
import struct

# Lecture du superbloc EXT
with open("disk.img", "rb") as f:
    f.seek(partition_offset + 1024)  # superbloc à +1024
    data = f.read(1024)

# Parse du nombre magique (unsigned short little-endian à l'offset 56)
magic = struct.unpack_from("<H", data, 56)[0]
if magic != 0xEF53:
    raise ValueError("Pas un système de fichiers EXT")

# Parse du nombre d'inodes (unsigned int little-endian à l'offset 0)
inode_count = struct.unpack_from("<I", data, 0)[0]

# Parse de la taille de bloc (calculée depuis la valeur log à l'offset 24)
log_block_size = struct.unpack_from("<I", data, 24)[0]
block_size = 1024 << log_block_size
```

Tous les champs sont en little-endian — FAT32 et EXT utilisent tous deux l'ordre little-endian, représenté par `<` dans les chaînes de format de struct Python.

---

## Exemple de sortie

```
=== Table de Partitions MBR ===
Partition 1 : type=0x0B (FAT32), start=2048, size=2097152 secteurs
Partition 2 : type=0x83 (Linux), start=2099200, size=20971520 secteurs

=== FAT32 BPB (Partition 1) ===
Label de volume   : MYUSB
Nom OEM           : MSDOS5.0
Octets/secteur    : 512
Secteurs/cluster  : 8
Taille de cluster : 4096 octets
Secteurs réservés : 32
Copies FAT        : 2
Total secteurs    : 2097152
Taille volume     : ~1,0 Go

=== Superbloc EXT (Partition 2) ===
Magique           : 0xEF53 ✓
Nom de volume     : linux-root
Taille de bloc    : 4096 octets
Nombre d'inodes   : 1310720
Nombre de blocs   : 2621440
État FS           : Propre
Fonctionnalités   : has_journal, extents, 64bit, flex_bg
Dernier montage   : /
```

---

## Créer des images de test

Pas besoin d'un vrai disque — on peut créer des images de test sous Linux en quelques secondes :

```bash
# Créer une image FAT32
dd if=/dev/zero of=fat32.img bs=1M count=64
mkfs.fat -F 32 fat32.img

# Créer une image EXT4
dd if=/dev/zero of=ext4.img bs=1M count=128
mkfs.ext4 ext4.img
```

Passez l'une ou l'autre image à FSParser et vous obtiendrez une décomposition complète des structures.

---

## Ce que j'ai appris

- Comment les tables de partitions MBR sont organisées et pourquoi l'alignement sur 512 octets est important
- La différence entre FAT12, FAT16 et FAT32 — et pourquoi la structure BPB diffère entre eux
- Pourquoi les nombres magiques EXT existent et comment le noyau les utilise au montage
- Que `struct.unpack` suffit pour parser des formats binaires si on a la spec
- La distinction entre EXT2, EXT3 et EXT4 au niveau du superbloc (bit de fonctionnalité journal, support des arbres d'extensions, adressage de blocs 64 bits)

---

## Systèmes de fichiers supportés

| Système de fichiers | Structures parsées | Notes |
|--------------------|-------------------|-------|
| FAT32 | BPB, entrée de partition | Extraction complète des métadonnées |
| EXT2/3/4 | Superbloc | Détection des fonctionnalités, info taille |
| MBR | Table de partitions (4 entrées) | Identification du type |
| GPT | — | Pas encore supporté |
| NTFS | — | Pas encore supporté |

---

## Ressources

- [Spécification du système de fichiers FAT32 (Microsoft)](https://download.microsoft.com/download/1/6/1/161ba512-40e2-4cc9-843a-923143f3456c/fatgen103.doc)
- [The Second Extended Filesystem — kernel.org](https://www.kernel.org/doc/html/latest/filesystems/ext2.html)
- [Documentation EXT4](https://www.kernel.org/doc/html/latest/filesystems/ext4/index.html)
- [Module Python struct](https://docs.python.org/3/library/struct.html)
- [OSDev wiki — MBR](https://wiki.osdev.org/MBR_(x86))
