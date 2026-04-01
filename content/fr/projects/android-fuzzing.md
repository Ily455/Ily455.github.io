---
title: "Fuzzing Android — Étude pratique"
date: 2023-05-01
draft: false
description: "Fuzzing automatisé de fichiers .dex Android avec Droid-FF et Radamsa sur émulateur Genymotion. Crashes confirmés dans libz.so, tombstones analysés, faute tracée jusqu'à adler32+227."
tags: ["fuzzing", "android", "droid-ff", "radamsa", "adb", "securite"]
---

> Projet académique — printemps 2023. Objectif : apprendre et pratiquer le fuzzing automatisé sur Android. Approche : utiliser Droid-FF pour générer des fichiers .dex mutés, les pousser sur un émulateur, exécuter dexdump, et analyser les crashs via le pipeline complet de triage.

---

## Environnement

**Hôte** : Kali Linux
**Émulateur** : Genymotion (image Samsung Galaxy S4 — 192.168.1.114:5555)
**Framework de fuzzing** : [Droid-FF](https://github.com/antojoseph/droid-ff)
**Moteur de mutation** : Radamsa
**Connexion** : ADB over TCP

L'émulateur est enregistré dans Droid-FF via `adb connect 192.168.1.114:5555`. Tout le transfert de fichiers et la collecte de crashs passent par ADB.

---

## Workflow

Droid-FF expose un menu en cinq étapes :

```
(0) Generate Files         — muter un fichier seed en N samples
(1) Start running fuzzer   — pousser les samples, exécuter la cible, logguer les crashs
(2) View Crashes           — récupérer logcat, identifier les entrées crashantes
(3) Triage Crashes         — confirmer la reproduction, collecter les tombstones
(4) View Source of Crashes — symboliser les adresses de crash
(5) Exploitability Test
```

### Étape 0 — Générer les fichiers

Radamsa choisi comme moteur de mutation, avec un fichier `.dex` de seed. **40 samples** générés.

### Étape 1 — Lancer le fuzzer

Pour chaque sample, Droid-FF :
1. Pousse le `.dex` dans `/data/local/tmp` via `adb push`
2. Exécute `adb shell /system/xbin/dexdump /data/local/tmp/sampleN.dex`
3. Logge les SIGSEGV depuis logcat

### Étape 2 — Voir les crashs

Logcat récupéré : **1375 lignes** au total. Plusieurs fichiers `.dex` ont déclenché SIGSEGV dans dexdump.

### Étape 3 — Triage des crashs

Pour chaque entrée crashante, Droid-FF :
1. Vide `/data/tombstones/*`
2. Re-pousse le fichier et relance dexdump
3. Confirme que le crash se reproduit
4. Tire le tombstone dans `fuzzer/confirmed_crashes/`

Exemple — `sample27.dex` :
```
adb push .../sample27.dex /data/local/tmp        → 0,9 MB/s, 553109 bytes
adb shell /system/xbin/dexdump /data/local/tmp/sample27.dex
→ tombstone_00 créé (10447 bytes, 2023-01-11 20:57)
adb pull /data/tombstones/tombstone_00 .../confirmed_crashes/tombstone_sample27.dex
→ 7,9 MB/s, 10447 bytes
```

### Étape 4 — Voir la source des crashs

L'option 4 directe a rencontré des erreurs. Utilisation de `dmesg` pour lire le buffer du tombstone :

```
signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0xf7609000
pid: 2600, tid: 2600, name: dexdump  >>> /system/xbin/dexdump <<<
ABI: 'x86'

eax f7609000  ebx 565a4f10  ecx 0002ff5f  edx 00000000
esi 00000023  edi 00000000

backtrace:
  #00 pc 000018b3  /system/lib/libz.so (adler32+227)
  #01 pc 0000d4db  /system/xbin/dexdump
  #02 pc 0000f0ff  /system/xbin/dexdump
  #03 pc 00006922  /system/xbin/dexdump
  #04 pc 00001b22  /system/xbin/dexdump
  #05 pc 00012a64  /system/lib/libc.so (__libc_init+100)
```

`libz.so` tiré depuis le device, puis addr2line :

```bash
adb pull /system/lib/libz.so
addr2line -f -e libz.so 000018b3
# → adler32
# → ??:?
```

La fonction est `adler32` dans `libz.so` à l'offset `0x18b3`. Les informations de ligne source sont absentes (binaire strippé, sans symboles de debug). Chaîne du crash : `.dex` malformé → décompression dexdump → adler32 dans libz → SIGSEGV à `0xf7609000`.

---

## Limites

**Émulateur vs matériel réel** : Genymotion utilise la virtualisation matérielle x86. Les vrais appareils Android tournent sur ARM. L'adresse de faute et l'état des registres diffèrent selon la plateforme — un crash sur émulateur ne garantit pas le même crash sur matériel.

**Absence de symboles de debug** : `addr2line` a identifié le nom de la fonction mais retourné `??:?` pour la ligne source — le libz.so du device est strippé. Pour aller plus loin, il faudrait un build AOSP avec les symboles de debug.

**Échec de l'option 4** : La vue source intégrée de Droid-FF a planté, nécessitant un fallback manuel via dmesg. Pas bloquant, mais une lacune dans le workflow de l'outil.

---

## Ce que j'en retiens

- Boucle complète de fuzzing mutatif : générer → pousser → exécuter → crasher → trier → symboliser
- Lecture des tombstones Android et mapping des adresses de crash vers les symboles avec addr2line
- Fonctionnement de Droid-FF pour coordonner ADB, détection de crashs et collecte de tombstones
- Limites de l'émulation pour l'analyse de crashs spécifiques au matériel

---

## Ressources

- [Droid-FF](https://github.com/antojoseph/droid-ff)
- [Radamsa](https://gitlab.com/akihe/radamsa)
- [Format des tombstones Android](https://source.android.com/docs/core/tests/debug/native-crash)
- [Documentation addr2line](https://man7.org/linux/man-pages/man1/addr2line.1.html)
