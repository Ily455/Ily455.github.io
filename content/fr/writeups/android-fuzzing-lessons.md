---
title: "Fuzzing Android — Méthodologie et retours pratiques"
date: 2023-05-01
draft: false
description: "Analyse approfondie de la méthodologie du projet de fuzzing Android avec Droid-FF : pipeline de triage, analyse des tombstones, et ce que le crash dans libz.so révèle vraiment."
tags: ["fuzzing", "android", "droid-ff", "radamsa", "adb", "recherche-securite"]
---

> Ce write-up approfondit la méthodologie et les choix d'analyse du projet de fuzzing Android. Pour le résumé des résultats et du setup, voir la [page projet](/projects/android-fuzzing/).

---

## Pourquoi les fichiers .dex

La cible de cette campagne de fuzzing est constituée de fichiers `.dex` traités par `dexdump` — pas de fichiers multimédia ni de la couche media. La raison : `dexdump` est un binaire système qui parse un format binaire structuré (Dalvik Executable), le charge en mémoire et utilise des bibliothèques système (dont `libz.so` pour la décompression). C'est une cible autonome accessible directement via ADB sans avoir à déclencher un comportement d'application — pas de système d'intent, pas de sandbox, pas de HAL.

Cela diffère des approches ciblant `mediaserver` ou `stagefright`, qui nécessitent de déclencher le parsing média via la couche IPC d'Android et qui ajoutent plusieurs points de défaillance. `dexdump` est accessible avec une seule commande `adb shell`.

---

## Le pipeline de triage

Le mécanisme de triage est au cœur de ce que Droid-FF apporte. La boucle générale de fuzzing est :

```
Générer → Pousser → Exécuter → Logger → Identifier les crashs → Confirmer → Symboliser
```

### Identifier les crashs depuis logcat

La sortie logcat est bruyante. Les processus système crashent indépendamment du fuzzing, et tous les signaux de crash arrivent dans le même flux. Droid-FF gère cela en inscrivant un marqueur avant l'exécution de chaque entrée — ce qui permet de corréler un SIGSEGV dans les logs à l'entrée qui l'a précédé.

En pratique, 1375 lignes de logcat ont été générées, avec des crashs visibles depuis plusieurs fichiers `.dex`. L'approche par marqueur rend cela parseable sans inspection manuelle ligne par ligne.

### Triage : reproduction et collecte de tombstones

L'option 3 relance uniquement les entrées crashantes :

1. Vider `/data/tombstones/*`
2. Pousser le sample crashant
3. Exécuter `dexdump` dessus
4. Vérifier si un tombstone a été créé
5. Si oui — le récupérer via `adb pull`

Le tombstone (`/data/tombstones/tombstone_00`) est le dump de crash d'Android. Il contient :
- Le signal et l'adresse de faute
- L'état des registres au moment du crash
- La backtrace complète avec les valeurs PC par frame
- Les descripteurs de fichiers ouverts et la carte mémoire

Pour `sample27.dex` : tombstone créé à 10 447 octets, récupéré dans `confirmed_crashes/tombstone_sample27.dex`.

### Symboliser le crash

L'option 4 (View Source of Crashes) a rencontré des erreurs dans le flux intégré de Droid-FF. Le fallback a été `dmesg` pour lire le buffer kernel du tombstone :

```
signal 11 (SIGSEGV), code 2 (SEGV_ACCERR), fault addr 0xf7609000
pid: 2600, name: dexdump >>> /system/xbin/dexdump <<<
ABI: 'x86'

backtrace:
  #00 pc 000018b3  /system/lib/libz.so (adler32+227)
  #01 pc 0000d4db  /system/xbin/dexdump
  ...
```

`SEGV_ACCERR` signifie "adresse présente en mémoire mais droits insuffisants" — contrairement à `SEGV_MAPERR` (adresse non mappée). Cela suggère une corruption de pointeur ou un dépassement de tampon qui a atteint une région protégée plutôt qu'un espace non mappé.

La backtrace localise la faute dans `libz.so` à `adler32+227`. Récupération de `libz.so` depuis le device et addr2line :

```bash
adb pull /system/lib/libz.so
addr2line -f -e libz.so 000018b3
# → adler32
# → ??:?
```

La fonction est identifiée mais la ligne source retourne `??:?` — le `libz.so` du device est strippé. Pour une résolution au niveau ligne, il faudrait un build de débogage d'AOSP correspondant à l'image du device.

---

## Ce que le chemin de crash révèle

Chaîne d'appel : `dexdump` → `libz.so` → `adler32+227`

`adler32` est une fonction de checksum utilisée par zlib pendant la décompression. Un fichier `.dex` muté par Radamsa peut avoir corrompu sa structure interne de façon à ce que `dexdump` tente de décompresser une section de données avec des paramètres invalides. `adler32` se trouve dans le chemin de vérification de la décompression — il calcule un checksum glissant sur le buffer décompressé pour vérifier l'intégrité. Une mutation qui corrompt les champs de longueur ou d'offset dans l'en-tête de section compressée du `.dex` pourrait amener `adler32` à opérer sur un buffer hors limites.

C'est un crash — pas une vulnérabilité exploitable confirmée. Le SIGSEGV à une adresse unique ne dit pas si l'entrée contrôle `eip` (nécessaire pour l'exécution de code). C'est l'objet de l'option 5 (Exploitability Test, via `gdb`/`gdbserver`) — non exécutée dans ce projet.

---

## Ce qui manquait

**dexRepair** : avant le fuzzing, Droid-FF exécute `dexRepair` sur chaque sample généré. Cela répare les champs structurels de l'en-tête `.dex` (checksums, offsets) pour que le fichier passe la validation initiale et atteigne le code de parsing plus profond. Sans cette étape, la plupart des `.dex` mutés seraient rejetés dès la vérification d'en-tête, avant d'atteindre la décompression de `libz.so`.

**Android 5.0** : la cible était Android 5.0 (Lollipop). Cela importe car le runtime ART (qui a remplacé Dalvik en 5.0) modifie la façon dont les `.dex` sont traités — et parce que le comportement de `libz.so` peut différer selon les versions Android.

**Cluster vs. appareil unique** : la conclusion du rapport de projet note qu'un cluster d'appareils Android aurait rendu la campagne significativement plus efficace. Avec une seule instance Genymotion, le débit était limité par la latence aller-retour ADB et l'exécution séquentielle des tests.

---

## Ce que j'en retiens

- Comment Droid-FF coordonne la boucle complète : génération, push, exécution, logging des crashs, triage, symbolisation
- Lecture des tombstones et mapping des adresses de crash avec `addr2line` — et pourquoi les binaires strippés limitent jusqu'où on peut aller
- `SEGV_ACCERR` vs `SEGV_MAPERR` comme signaux de crash différents avec des implications différentes
- Pourquoi `dexRepair` est important : les sorties du fuzzer doivent survivre à la validation du format avant d'atteindre le code de parsing intéressant
- L'écart entre "crash confirmé" et "exploitable" — c'est l'étape du test d'exploitabilité

---

## Ressources

- [Droid-FF](https://github.com/antojoseph/droid-ff)
- [Format des tombstones Android](https://source.android.com/docs/core/tests/debug/native-crash)
- [Source adler32 de zlib](https://github.com/madler/zlib/blob/master/adler32.c)
- [Documentation addr2line](https://man7.org/linux/man-pages/man1/addr2line.1.html)
