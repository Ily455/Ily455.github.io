---
title: "Forensics Mémoire avec Volatility — Analyse d'un Dump WinXP SP2"
date: 2024-06-01
draft: false
description: "Walkthrough complet d'un dump mémoire Windows XP SP2 avec Volatility 2.6.1 — 14 plugins couvrant l'énumération des processus, la détection de processus cachés, les connexions réseau, les artéfacts registre, les modules noyau et l'inspection mémoire live via volshell."
tags: ["forensics", "volatility", "forensics-memoire", "windows", "dfir", "blue-team"]
---

> Lab réalisé dans le cadre de mon M2 SICS (2024). Objectif : pratiquer la forensics mémoire basique et avancée sur un dump WinXP SP2 — de l'identification du profil à l'inspection interactive avec volshell. Toutes les commandes sont en syntaxe Volatility 2.

---

## Cible

```
Fichier  : dump_practice.dmp
Profil   : WinXPSP2x86
Capture  : 2016-01-03 23:00:28 UTC
Service Pack : 2
Nombre de processeurs : 1
KDBG : 0x8054c060L
```

Le dump a été capturé le 2016-01-03, mais la plupart des processus ont démarré le 2015-12-23 — la machine tournait depuis ~11 jours au moment de la capture.

---

## 1. Identification du profil — `imageinfo`

```bash
volatility -f /dumps/dump_practice.dmp imageinfo
```

```
Suggested Profile(s) : WinXPSP2x86, WinXPSP3x86 (Instantiated with WinXPSP2x86)
AS Layer1 : IA32PagedMemory (Kernel AS)
AS Layer2 : VirtualBoxCoreDumpElf64 (Unnamed AS)
AS Layer3 : FileAddressSpace (/dumps/dump_practice.dmp)
PAE type  : No PAE
DTB       : 0x39000L
KDBG      : 0x8054c060L
Image date and time : 2016-01-03 23:00:28 UTC+0000
```

`AS Layer2` indique `VirtualBoxCoreDumpElf64` — le dump provient d'une VM VirtualBox. `imageinfo` affine le profil à `WinXPSP2x86`, utilisé pour toutes les commandes suivantes.

---

## 2. Liste des processus — `pslist`

```bash
volatility -f /dumps/dump_practice.dmp pslist
```

`pslist` parcourt la liste doublement chaînée `PsActiveProcessHead` du noyau — la liste officielle des processus de l'OS. Extrait :

```
Offset(V)  Nom               PID   PPID  Thds  Hnds  Démarrage
0x823c89c8 System              4      0    51   249   —
0x821b4020 smss.exe          492      4     4     3   2015-12-23 09:25:39
0x8229f020 csrss.exe         560    492    13   390   2015-12-23 09:25:39
0x8217b510 winlogon.exe      584    492    16   418   2015-12-23 09:25:39
0x82176b28 services.exe      628    584    16   263   2015-12-23 09:25:39
0x820fdda0 explorer.exe     1596   1568    12   379   2015-12-23 00:25:42
0x8230ada0 mspaint.exe      1772   1596     4    98   2015-12-23 01:39:56
0x820095c8 wmplayer.exe     1360   1776    29   701   2015-12-23 00:31:30
0x82077020 notepad.exe      1260   1596     0   ——    2015-12-23 01:20:49 (terminé 01:40:40)
0x81f819c8 notepad.exe      1788   1596     0   ——    2015-12-23 01:42:15 (terminé 2016-01-03 22:55:04)
0x81fc8020 notepad.exe      1088   1596     1    27   2016-01-03 22:56:02
```

À noter : 4 instances de notepad.exe — 3 déjà terminées, 1 (PID 1088) encore active au moment de la capture. `wmplayer.exe` PID 1360 a un parent inhabituel (PID 1776, `wpabaln.exe`).

---

## 3. Détection de processus cachés — `psscan`

```bash
volatility -f /dumps/dump_practice.dmp psscan
```

`psscan` scanne la mémoire physique brute à la recherche de structures `_EPROCESS` par signature — sans passer par la liste chaînée maintenue par l'OS. Cela peut révéler des processus cachés par des rootkits qui se sont décrochés de `PsActiveProcessHead`.

Résultat : dans ce dump, `psscan` et `pslist` donnent exactement la même liste. Aucun processus caché détecté.

C'est la vérification forensique clé : si `psscan` révèle un processus absent de `pslist`, il a été activement dissimulé — fort indicateur de rootkit.

---

## 4. Handles d'un processus — `handles`

```bash
volatility -f /dumps/dump_practice.dmp handles -p 1772
```

Liste tous les handles noyau ouverts par `mspaint.exe` (PID 1772). Les handles sont des références à des ressources OS — fichiers, clés de registre, événements, mutexes, etc. :

```
Offset(V)   Pid  Handle  Type             Détails
0xe10096a0 1772  0x4     KeyedEvent       CritSecOutOfMemoryEvent
0x81fffbb0 1772  0xc     File             \Device\HarddiskVolume1\Documents and Settings\pero
0x8214b810 1772  0x1c    File             \Device\HarddiskVolume1\WINDOWS\WinSxS\x86_Microsoft.Windows.Common-Controls_...
0x8215b810 1772  0x28    WindowStation    WinSta0
0x82179038 1772  0x2c    Desktop          Default
0xe1129550 1772  0x34    Key              MACHINE
0x82166f38 1772  0x38    Semaphore        shell.{A48F1A32-A340-11D1-BC6B-00A0C90312E1}
0x81fd8ca0 1772  0x40    File             \Device\KsecDD
0xe193e0f0 1772  0x50    Key              MACHINE\SOFTWARE\MICROSOFT\WINDOWS NT\CURRENTVERSION\DRIVERS32
```

Le handle de fichier ouvert sur `\Documents and Settings\pero` indique le profil utilisateur actif. `\Device\KsecDD` est le fournisseur de support de sécurité du noyau — présent dans la plupart des processus GUI pour les opérations liées aux credentials.

---

## 5. Artéfacts d'exécution registre — `userassist`

```bash
volatility -f /dumps/dump_practice.dmp userassist
```

`userassist` extrait les clés UserAssist de Windows — un historique d'exécution encodé en ROT13, stocké dans `NTUSER.DAT`. Ces clés enregistrent les programmes lancés par l'utilisateur, leur fréquence, et la date de dernier lancement.

```
Registre : \Device\HarddiskVolume1\Documents and Settings\pero\NTUSER.DAT
Chemin   : Software\Microsoft\Windows\CurrentVersion\Explorer\UserAssist\{75048700-...}\Count

UEME_RUNPIDL:%csidl2%\MSN.lnk
  Count        : 14
  Dernière màj : 2015-12-23 00:22:13 UTC+0000

UEME_RUNPIDL:%csidl2%\Windows Media Player.lnk
  Count        : 14
  Dernière màj : 2015-12-23 00:31:20 UTC+0000

UEME_RUNPIDL:%csidl2%\Accessories\Notepad.lnk
  Count        : 1
  Dernière màj : 2016-01-03 22:56:02 UTC+0000
```

L'utilisateur `pero` a lancé MSN et Windows Media Player 14 fois chacun lors de la première session (2015-12-23). Notepad a été lancé une seule fois le 2016-01-03 — correspondant exactement à `notepad.exe` PID 1088 encore actif dans pslist.

Le GUID `{75048700-EF1F-11D0-9888-006097DEACF9}` correspond à la classe `UEME_RUNPIDL` (éléments lancés depuis le menu Démarrer via raccourcis).

---

## 6. Modules noyau — `modules`

```bash
volatility -f /dumps/dump_practice.dmp modules
```

`modules` parcourt `PsLoadedModuleList` pour énumérer les modules en mode noyau (drivers, le kernel lui-même). Ils s'exécutent avec les privilèges complets ring-0 :

```
Offset(V)   Nom            Base        Taille  Fichier
0x823fc3a0  ntoskrnl.exe   0x804d7000  0x214200  \WINDOWS\system32\ntoskrnl.exe
0x823fc338  hal.dll        0x806ec000  0x13d80   \WINDOWS\system32\hal.dll
0x823fc2d0  kdcom.dll      0xf8a50000  0x2000    \WINDOWS\system32\KDCOM.DLL
0x823fc1f8  ACPI.sys       0xf8501000  0x2e000   ACPI.sys
0x823fc188  WMILIB.SYS     0xf8a52000  0x2000    \WINDOWS\system32\DRIVERS\WMILIB.SYS
0x823fc120  pci.sys        0xf84f0000  0x11000   pci.sys
```

Les rootkits chargent souvent des modules noyau ou cachent des modules existants de cette liste. La comparaison `modules` / `driverscan` permet de détecter des drivers dissimulés via manipulation de `PsLoadedModuleList`.

---

## 7. Scan des drivers — `driverscan`

```bash
volatility -f /dumps/dump_practice.dmp driverscan
```

`driverscan` scanne la mémoire physique à la recherche de structures `_DRIVER_OBJECT` par pool tag — indépendamment de la liste chaînée des modules. Les différences entre `driverscan` et `modules` indiquent des drivers cachés.

Driver notable trouvé : `PROCEXP141` — le driver noyau de Process Explorer, ce qui correspond à `procexp.exe` (PID 1204) dans la liste des processus. Process Explorer tournait avec son driver noyau chargé.

```
0x0000000021756c0  3  0 0xf8908000  0x4380 PROCEXP141  PROCEXP141  \Driver\PROCEXP141
```

---

## 8. Connexions réseau — `connections`

```bash
volatility -f /dumps/dump_practice.dmp connections
```

```
Offset(V)   Adresse locale     Adresse distante  Pid
0x81feed00  10.0.2.15:1242     50.31.192.83:554  1360
```

Une seule connexion TCP active : port local 1242 → port distant **554** (RTSP — Real Time Streaming Protocol). PID 1360 est `wmplayer.exe`. Windows Media Player diffusait du contenu depuis `50.31.192.83` en RTSP — cohérent avec le nombre élevé de handles (701) et les 14 entrées UserAssist de WMP.

Le port 554 RTSP est le port standard pour les serveurs de streaming. Dans un binaire suspect, ce serait un indicateur C2 significatif.

---

## 9. Historique IE — `iehistory`

```bash
volatility -f /dumps/dump_practice.dmp iehistory
```

```
Process: 1596 explorer.exe
Cache type "DEST" at 0x15ceef
URL: pero@http://sc1.slable.com:8126
Last accessed: 2015-12-23 01:23:20 UTC+0000

Process: 1596 explorer.exe
Location: Visited: pero@about:Home
Location: Visited: pero@res://C:\WINDOWS\system32\shdoclc.dll\dnserror.htm
```

L'utilisateur `pero` avait IE ouvert sur une URL sur `sc1.slable.com:8126` et a visité la page d'erreur DNS d'IE (`dnserror.htm`) — qui apparaît généralement quand la résolution DNS échoue ou que le réseau est indisponible.

---

## 10. Historique console — `consoles`

```bash
volatility -f /dumps/dump_practice.dmp consoles
```

```
ConsoleProcess: csrss.exe Pid: 560
Console: 0x4e23b0 CommandHistorySize: 50
HistoryBufferCount: 1 HistoryBufferMax: 4
Title: ??\WINDOWS\system32\cmd.exe
```

Une session `cmd.exe` était ouverte (gérée par `csrss.exe`). La taille du buffer de commandes est 50 mais seulement 1 buffer d'historique actif — la console était ouverte mais peu de commandes ont été saisies. Le contenu exact des commandes n'a pas pu être récupéré dans ce dump.

---

## 11. Inspection des threads — `threads`

```bash
volatility -f /dumps/dump_practice.dmp threads -p 1088
```

`threads` énumère les structures `_ETHREAD` du noyau pour le PID 1088 (`notepad.exe`) :

```
ETHREAD: 0x81f99da8  Pid: 1088  Tid: 1488
Créé     : 2016-01-03 22:56:02 UTC+0000
Processus: notepad.exe
État     : Waiting:WrUserRequest
Priorité : 0x8
StartAddress : 0x7c810867 kernel32.dll
```

L'état `WrUserRequest` signifie que le thread est bloqué en attente d'une entrée utilisateur — exactement ce qu'on attend d'une fenêtre Notepad ouverte et inactive. L'adresse de départ dans `kernel32.dll` est le point standard de création de threads Win32.

---

## 12. Inspection mémoire interactive — `volshell`

```bash
volatility -f /dumps/dump_practice.dmp volshell -p 1088
```

`volshell` fournit un shell Python interactif avec accès complet à l'espace d'adressage du processus. Contexte courant : `notepad.exe @ 0x81fc8020, pid=1088`.

**Désassemblage à l'EIP :**

```python
>>> dis(0x7c90eb94)
0x7c90eb94 c3          RET
0x7c90eb95 8da42400000000  LEA ESP, [ESP+0x0]
...
0x7c90ebac 55          PUSH EBP
0x7c90ebad 8bec        MOV EBP, ESP
0x7c90ebaf 9c          PUSHF
0x7c90ebb0 81ecd0020000    SUB ESP, 0x2d0
...
0x7c90eba9 cd2e        INT 0x2e
0x7c90ebab c3          RET
```

Territoire `kernel32.dll` — le désassemblage montre un `RET`, un NOP sled, puis un prologue de fonction complet. L'instruction `INT 0x2e` est la porte des appels système de Windows XP (remplacée ensuite par `SYSENTER`).

**Dump mémoire à la pile (ESP) :**

```python
>>> dd(0x0007febc)
0007febc  77d4919b 77d491ce 0007fefc 00000000
0007fecc  00000000 00000000 00000000 0007ff1c
0007fedc  01002a1b 0007fefc 00000000 00000000
0007feec  00000000 00000000 7c80b529 000a2332
```

Les données de la pile à ESP montrent des adresses de retour dans la plage de `ntdll.dll` (`0x77xxxxxx`) et `kernel32.dll` (`0x7c8xxxxx`).

---

## 13. Extraction de texte Notepad — `editbox`

```bash
volatility -f /dumps/dump_practice.dmp editbox -p 1088
```

`editbox` lit directement le buffer du contrôle Win32 edit depuis l'état GUI du processus :

```
Wnd Context    : 0\WinSta0\Default
Process ID     : 1088
ImageFileName  : notepad.exe
nChars         : 17
undoBuf        : Mhm.

You are the best.
```

Puis via le plugin `notepad` pour toutes les instances :

```bash
volatility -f /dumps/dump_practice.dmp notepad
Process: 1088
Text:
You are the best. Mhm. g
```

La fenêtre Notepad active (PID 1088) contenait le texte **"You are the best. Mhm. g"**. Le buffer undo contenait "Mhm." — la dernière opération d'édition était l'ajout de " Mhm." au texte. Extraction directe depuis la mémoire du processus, sans accès au fichier.

---

## 14. Virtual Address Descriptors — `vadinfo`

```bash
volatility -f /dumps/dump_practice.dmp vadinfo -p 1088
```

Les VADs décrivent la disposition de la mémoire virtuelle d'un processus — quelles régions sont privées, mappées depuis un fichier, ou des sections image :

```
VAD node @ 0x82167318  Start 0x00020000  End 0x00020fff  Tag VadS
  Flags: CommitCharge: 1, MemCommit: 1, PrivateMemory: 1, Protection: 4
  Protection: PAGE_READWRITE

VAD node @ 0x8200f7a8  Start 0x01000000  End 0x01013fff  Tag VadI
  Flags: CommitCharge: 3, ImageMap: 1, Protection: 7
  Protection: PAGE_EXECUTE_WRITECOPY
  FileObject @ 820759e0, Name: \Device\HarddiskVolume1\WINDOWS\system32\notepad.exe
```

`VadS` (régions pile/tas privées) ont `PAGE_READWRITE`. L'entrée `VadI` à `0x01000000` est l'image mappée de `notepad.exe` — `PAGE_EXECUTE_WRITECOPY` est la protection standard pour les sections image exécutables sous Windows.

L'inspection VAD est utile pour détecter l'injection de processus : une région `PAGE_EXECUTE_READWRITE` sans image mappée correspondante est un fort indicateur de shellcode.

---

## Bilan

| Constat | Plugin | Signification |
|---------|--------|---------------|
| Profil : WinXPSP2x86 | `imageinfo` | Base pour toutes les commandes |
| Aucun processus caché | `psscan` vs `pslist` | Pas de rootkit actif dans ce dump |
| WMP → flux RTSP vers 50.31.192.83:554 | `connections` | PID 1360, streaming légitime |
| Utilisateur "pero", 14× WMP, 1× Notepad | `userassist` | Reconstruction de la timeline d'exécution |
| Driver PROCEXP141 chargé | `driverscan` | Process Explorer tournait avec son driver noyau |
| Texte Notepad : "You are the best. Mhm." | `editbox` / `notepad` | Extraction depuis la mémoire live sans accès fichier |
| VAD notepad.exe mappé à 0x01000000 | `vadinfo` | Mapping image normal, aucune injection détectée |

La comparaison pslist/psscan est la technique centrale de détection de processus cachés. L'extraction editbox démontre que la forensics mémoire peut récupérer le contenu d'un document même si le fichier n'a jamais été sauvegardé sur disque.

---

## Ressources

- [Volatility 2 Command Reference](https://github.com/volatilityfoundation/volatility/wiki/Command-Reference)
- [Windows Process Internals — Memory Forensics (imphash)](https://imphash.medium.com/windows-process-internals-a-few-concepts-to-know-before-jumping-on-memory-forensics-part-4-16c47b89e826)
- [The Art of Memory Forensics — Ligh, Case, Levy, Walters](https://www.memoryanalysis.net/)
- [Volatility Foundation](https://volatilityfoundation.org/)
