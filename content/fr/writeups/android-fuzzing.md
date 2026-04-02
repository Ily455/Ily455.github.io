---
title: "Fuzzing Android avec Droid-FF et Radamsa"
date: 2023-05-01
draft: false
description: "Fuzzing automatisé de fichiers .dex Android avec Droid-FF et Radamsa sur émulateur Genymotion. Pipeline de triage complet : détection de crashs via logcat, collecte de tombstones, symbolisation addr2line, et analyse du chemin de crash à travers adler32 dans libz.so."
tags: ["fuzzing", "android", "droid-ff", "radamsa", "adb", "securite"]
---

> Projet académique — printemps 2023. Objectif : apprendre et pratiquer le fuzzing automatisé sur Android. Cible : fichiers `.dex` traités par `dexdump`. Framework : Droid-FF avec Radamsa comme moteur de mutation.

---

## Environnement

**Hôte** : Kali Linux
**Émulateur** : Genymotion (image Samsung Galaxy S4 — 192.168.1.114:5555)
**Framework de fuzzing** : [Droid-FF](https://github.com/antojoseph/droid-ff)
**Moteur de mutation** : Radamsa
**Connexion** : ADB over TCP

L'émulateur est enregistré dans Droid-FF via `adb connect 192.168.1.114:5555`. Tout le transfert de fichiers et la collecte de crashs passent par ADB.

---

## Pourquoi les fichiers .dex

La cible de cette campagne est constituée de fichiers `.dex` traités par `dexdump` — pas de fichiers multimédia ni de la couche media. `dexdump` est un binaire système qui parse un format binaire structuré (Dalvik Executable), le charge en mémoire et utilise des bibliothèques système (dont `libz.so` pour la décompression). C'est une cible autonome accessible avec une seule commande `adb shell` — pas de système d'intent, pas de sandbox, pas de HAL.

Cela diffère des approches ciblant `mediaserver` ou `stagefright`, qui nécessitent de déclencher le parsing média via la couche IPC d'Android et ajoutent plusieurs points de défaillance.

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

Radamsa choisi comme moteur de mutation avec un fichier `.dex` de seed. **50 samples** générés.

### Étape 1 — Lancer le fuzzer

Pour chaque sample, Droid-FF :
1. Exécute `dexRepair` sur le sample pour qu'il soit accepté comme `.dex` valide par la cible
2. Pousse le `.dex` dans `/data/local/tmp` via `adb push`
3. Exécute `adb shell /system/xbin/dexdump /data/local/tmp/sampleN.dex`
4. Logge les SIGSEGV depuis logcat
5. Supprime le fichier testé du device

**Note sur dexRepair** : avant le fuzzing, Droid-FF répare les champs structurels de l'en-tête `.dex` (checksums, offsets) pour que le fichier passe la validation initiale et atteigne le code de parsing plus profond. Sans cette étape, la plupart des `.dex` mutés seraient rejetés dès la vérification d'en-tête, avant d'atteindre la décompression dans `libz.so`.

### Étape 2 — Voir les crashs

Logcat récupéré : **1375 lignes** au total. Plusieurs fichiers `.dex` ont déclenché SIGSEGV dans dexdump.

La sortie logcat est bruyante — les processus système crashent indépendamment du fuzzing et tous les signaux de crash arrivent dans le même flux. Droid-FF gère cela en inscrivant un marqueur avant chaque entrée testée, permettant de corréler crash et entrée sans inspection manuelle.

### Étape 3 — Triage des crashs

Pour chaque entrée crashante, Droid-FF :
1. Vide `/data/tombstones/*`
2. Re-pousse le fichier et relance dexdump
3. Confirme que le crash se reproduit
4. Tire le tombstone dans `fuzzer/confirmed_crashes/`

Le tombstone (`/data/tombstones/tombstone_00`) est le dump de crash d'Android. Il contient le signal et l'adresse de faute, l'état des registres au moment du crash, la backtrace complète avec les valeurs PC par frame, ainsi que les descripteurs de fichiers ouverts et la carte mémoire.

Exemple — `sample27.dex` :
```
adb push .../sample27.dex /data/local/tmp        → 0,9 MB/s, 553109 bytes
adb shell /system/xbin/dexdump /data/local/tmp/sample27.dex
→ tombstone_00 créé (10447 bytes, 2023-01-11 20:57)
adb pull /data/tombstones/tombstone_00 .../confirmed_crashes/tombstone_sample27.dex
→ 7,9 MB/s, 10447 bytes
```

### Étape 4 — Symboliser le crash

L'option 4 directe a rencontré des erreurs. Utilisation de `dmesg` pour lire le buffer kernel du tombstone :

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

---

## Ce que le chemin de crash révèle

`SEGV_ACCERR` (code 2) signifie "adresse présente en mémoire mais droits insuffisants" — à distinguer de `SEGV_MAPERR` (adresse non mappée). Cela suggère une corruption de pointeur ou un dépassement de tampon ayant atteint une région protégée.

La backtrace localise la faute dans `libz.so` à `adler32+227`. Chaîne d'appel : `dexdump` → `libz.so` → `adler32+227`.

`adler32` est une fonction de checksum utilisée par zlib pendant la décompression. Un fichier `.dex` muté par Radamsa peut avoir corrompu sa structure interne de façon à ce que `dexdump` tente de décompresser une section avec des paramètres invalides. Une mutation qui corrompt les champs de longueur ou d'offset dans l'en-tête de section compressée du `.dex` pourrait amener `adler32` à opérer sur un buffer hors limites.

La fonction est identifiée, mais la ligne source retourne `??:?` — le `libz.so` du device est strippé. Une résolution au niveau ligne nécessiterait un build de débogage AOSP correspondant à l'image du device.

**C'est un crash — pas une vulnérabilité exploitable confirmée.** Le SIGSEGV ne dit pas si l'entrée contrôle `eip`. C'est l'objet de l'option 5 (Exploitability Test via `gdb`/`gdbserver`) — non exécutée dans ce projet.

---

## Limites

**Émulateur vs matériel réel** : Genymotion utilise la virtualisation matérielle x86. Les vrais appareils Android tournent sur ARM. L'adresse de faute et l'état des registres diffèrent selon la plateforme — un crash sur émulateur ne garantit pas le même crash sur matériel.

**Android 5.0** : la cible était Android 5.0 (Lollipop). Le runtime ART (qui a remplacé Dalvik en 5.0) modifie la façon dont les `.dex` sont traités, et le comportement de `libz.so` peut différer selon les versions Android.

**Absence de symboles de debug** : `addr2line` a identifié le nom de la fonction mais retourné `??:?` pour la ligne source. Aller plus loin nécessiterait un build AOSP avec symboles de debug.

**Échec de l'option 4** : la vue source intégrée de Droid-FF a planté, nécessitant un fallback manuel via `dmesg`.

**Débit sur device unique** : un cluster d'appareils Android aurait rendu la campagne significativement plus efficace. Avec une seule instance Genymotion, le débit était limité par la latence aller-retour ADB et l'exécution séquentielle des tests.

---

## Ce que j'en retiens

- Boucle complète de fuzzing mutatif : générer → pousser → exécuter → crasher → trier → symboliser
- Lecture des tombstones Android et mapping des adresses de crash avec addr2line
- `SEGV_ACCERR` vs `SEGV_MAPERR` comme signaux de crash différents avec des implications différentes
- Pourquoi `dexRepair` est important : les sorties du fuzzer doivent survivre à la validation du format avant d'atteindre le code de parsing intéressant
- Fonctionnement de Droid-FF pour coordonner ADB, détection de crashs et collecte de tombstones
- L'écart entre "crash confirmé" et "exploitable" — c'est l'étape du test d'exploitabilité
- Limites de l'émulation pour l'analyse de crashs spécifiques au matériel

---

## Ressources

- [Droid-FF](https://github.com/antojoseph/droid-ff)
- [Radamsa](https://gitlab.com/akihe/radamsa)
- [Format des tombstones Android](https://source.android.com/docs/core/tests/debug/native-crash)
- [Source adler32 de zlib](https://github.com/madler/zlib/blob/master/adler32.c)
- [Documentation addr2line](https://man7.org/linux/man-pages/man1/addr2line.1.html)
