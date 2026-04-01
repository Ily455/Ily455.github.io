---
title: "Étude d'Obfuscation LLVM — Stage Secure-IC"
date: 2024-07-01
draft: false
description: "Analyse comparative de 4 obfuscateurs LLVM (O-LLVM, Hikari, Pluto, Tigress) sur du code AES, suivie d'une intégration PoC de Hikari dans la chaîne de compilation du firmware Securyzr iSE de Secure-IC."
tags: ["llvm", "obfuscation", "reverse-engineering", "embarqué", "aes", "risc-v", "secure-ic"]
---

> Stage de fin d'études chez **Secure-IC** (Rennes, fév.–juil. 2024). Équipe : HOST SW Agent Program, département Chip to Cloud. Encadrant : Sofiane TAKARABT.

---

## Contexte

Le **Securyzr iSE** de Secure-IC est un module de sécurité matériel (HSM) basé sur un processeur RISC-V. Il gère les opérations cryptographiques, le démarrage sécurisé, le provisionnement de clés et l'identité des appareils pour les marchés IoT/automobile/défense. Le firmware qui y tourne contient de la propriété intellectuelle sensible.

Objectif : évaluer des outils d'obfuscation, en sélectionner un, et l'intégrer dans le système de build de Securyzr pour protéger le code firmware sensible contre le reverse engineering.

---

## Analyse Comparative

### Environnement

Quatre VMs Ubuntu Server 22.04.4 dans VirtualBox 7.0 sur un hôte Windows 10 (Dell, i5-1135G7, 16 Go RAM). Chaque VM avait 3072 Mo de RAM et 4 vCPUs — une par obfuscateur, pour éviter les conflits de dépendances.

| Obfuscateur | Type | Version LLVM |
|-------------|------|-------------|
| O-LLVM | Plugin LLVM | 4.0.1 |
| Hikari | Plugin LLVM | 15.0.2 |
| Pluto | Plugin LLVM | 14.0.6 |
| Tigress | Source-à-source | — |

Code de test : `prod_mat.c` (multiplication matricielle) et `aes.c` (implémentation TinyAES).

Outils de mesure : `clock()` pour le temps d'exécution, `du` pour la taille des binaires, `heaptrack` + `/bin/time` pour la mémoire, BinDiff + Ghidra (BinExport) pour la similarité binaire.

### Transformations Testées

| Transformation | Description |
|----------------|-------------|
| BCF | Bogus Control Flow — insère des blocs factices gardés par des prédicats opaques |
| FLA | Aplatissement du flot de contrôle — route tous les blocs via un dispatcher |
| SUB | Substitution d'instructions — remplace les opérateurs par des séquences équivalentes |
| MBA | Arithmétique Booléenne Mixte |
| IB | Branchement Indirect |
| FW | Fonction Wrapper |
| BBS | Découpage de Blocs de Base |
| GLE | Chiffrement des Variables Globales |
| VRTLZ | Virtualisation (Tigress uniquement) — convertit une fonction en interpréteur bytecode |
| TWO | BCF + FLA combinés |

### Impact sur la Taille des Binaires

BCF et FLA introduisent le plus de surcoût. Tigress VRTLZ est extrême :

| Obfuscateur | Transformation | aes.c original | aes.c obfusqué | Ratio |
|-------------|----------------|----------------|----------------|-------|
| Tigress | VRTLZ | 18 672 o | 2 646 922 o | **61,2×** |
| Tigress | BCF | 47 960 o | 161 986 o | 3,75× |
| Pluto | TWO | 25 344 o | 89 848 o | 3,5× |
| O-LLVM | TWO | 25 376 o | 47 584 o | 1,88× |
| Hikari | SE | 25 440 o | 62 504 o | 2,46× |

SUB, FW, MBA et SE ont un impact minimal sur la taille pour tous les outils.

### Temps d'Exécution

O-LLVM avait le pire surcoût d'exécution. Hikari était le plus efficace :

- O-LLVM FLA sur `prod_mat.c` : ratio ~5×
- O-LLVM TWO sur `prod_mat.c` : ratio ~22×
- Tigress TWO sur `aes.c` : 0,9013 s (exclu comme valeur aberrante — 50× la baseline)
- Tigress BCF sur `prod_mat.c` : **segmentation fault** — le binaire crashait
- Hikari : constamment sous 4× de surcoût sur toutes les transformations

### Scores de Similarité (BinDiff)

Score plus bas = plus grande divergence par rapport à l'original = obfuscation plus forte :

| Obfuscateur | Transformation | Similarité |
|-------------|----------------|------------|
| Pluto | BCF | **31,4 %** |
| Tigress | BCF | 34,3 % |
| O-LLVM | BCF | 55,4 % |
| Hikari | BCF | 77,9 % |
| Tous | SUB | ~47–48 % |
| Hikari | IB | 49,2 % |

BCF est la transformation la plus impactante pour tous les outils. Pluto produisait les scores les plus bas (plus obfusqué), mais avec un surcoût d'exécution plus élevé qu'Hikari.

### Comparaison Compilateurs : LLVM (Clang 14.0) vs GCC 11.4

- Clang prend légèrement plus de temps à compiler à tous les niveaux d'optimisation
- Les deux compilateurs améliorent le temps d'exécution avec des niveaux `-O` plus élevés
- Les binaires compilés par Clang consomment significativement moins de mémoire que GCC à partir de `-O1`

### Analyse d'Impact

Un pattern clair : un score de similarité plus bas corrèle avec un coût en ressources plus élevé (taille + temps d'exécution). Les transformations comme SUB, FW, MBA, SE produisent peu de changements sur le CFG et affectent à peine les performances. BCF et FLA modifient significativement la structure binaire mais à un coût en ressources.

**Hikari** offre le meilleur compromis : obfuscation significative avec un surcoût acceptable, basé sur la version LLVM la plus récente, et sans les crashs observés avec Tigress.

---

## PoC — Intégration dans Securyzr iSE

### Obfuscateur Retenu : Hikari

Choisi pour : le plus large ensemble de transformations parmi les plugins LLVM, base LLVM 15.0.2 (la plus récente), meilleure performance en temps d'exécution, et aucun problème de stabilité (contrairement à Tigress qui crashait sur plusieurs cas de test).

### Serveur de Build

| Spécification | Valeur |
|---------------|--------|
| CPU | Intel Xeon Gold 6242R @ 3,10 GHz |
| RAM | 64 Go |
| Stockage | >2,5 To |
| OS | Red Hat Enterprise Linux 8.5 |

### Compilation de Hikari

```bash
cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release \
  -DLLVM_ENABLE_PROJECTS=clang \
  -DLLVM_TARGETS_TO_BUILD="X86;ARM;RISCV;AArch64" \
  ./llvm -B ./build
ninja
```

Build multi-architectures ciblant X86, ARM, RISC-V et AArch64 — pour correspondre à la cible RISC-V de Securyzr et à l'environnement de développement x86.

### Code Cible

Le Securyzr iSE tourne sur RISC-V et gère la génération de la NVM (Non-Volatile Memory) au pré-boot. La NVM contient le firmware qui doit être chiffré avant d'être envoyé au Secure Element pour authentification. Le chiffrement utilise une implémentation propriétaire d'AES GCM en C — c'est ce code qui est ciblé par l'obfuscation.

Le périmètre a été limité au sous-ensemble AES (compilable x86, sans cross-compilation) compte tenu des 6 mois du stage.

### Intégration dans le Système de Build

L'intégration a nécessité trois modifications à la hiérarchie de Makefiles existante :

1. **Remplacement du compilateur** : passer de `gcc` au clang avec Hikari intégré
2. **Flags compilateur** : ajouter les flags `-mllvm -<transformation>` pour chaque technique d'obfuscation
3. **Sortie IR** : configurer le build pour émettre des fichiers `.ll`/`.bc` en plus des `.o`, permettant une recompilation multi-architecture ultérieure

```makefile
CC = /path/to/hikari/bin/clang
CFLAGS_IR = -emit-llvm -c -mllvm -bcf  # exemple : Bogus Control Flow
```

Le build produit un répertoire `build/` avec :
- `obj/` : fichiers objets obfusqués
- `ir/` : fichiers LLVM IR (indépendants de la plateforme, réutilisables)
- `bin/` : exécutable final

### Validation

Deux tests sur le matériel FPGA cible (CPUs ARM + RISC-V) :

- **Test `prog`** — lance le flux de démarrage sécurisé, qui nécessite que la NVM soit correctement générée et chargée. Résultat : NVM générée, transférée, firmware lancé avec succès.
- **`CRYPTO_TESTS`** — teste les fonctions cryptographiques du SE. Résultat : le code AES obfusqué a produit les sorties correctes.

Le code obfusqué n'a eu aucun impact fonctionnel sur le processus de build ou la séquence de boot Securyzr.

---

## Difficultés Rencontrées

- **Compatibilité toolchain** : changer de compilateur dans un projet C avec une hiérarchie de makefiles a nécessité de résoudre des problèmes de compatibilité de linker et de bibliothèques
- **Cross-compilation** : Securyzr cible RISC-V ; le serveur de développement est x86. L'approche IR-first (émettre `.ll`, recompiler pour la cible) était la voie à suivre mais impliquait une complexité du système de build
- **Stabilité** : Tigress a généré des crashs lors de l'analyse — ce qui l'a exclu pour la production quelle que soit sa force d'obfuscation
- **Limites de mesure** : `heaptrack` montrait 76,8 Ko uniforme pour toutes les transformations (probablement dû à l'allocation statique), nécessitant `/bin/time` comme solution de secours

---

## Ce que j'en retiens

- Comment comparer des outils d'obfuscation de façon systématique — taille, temps d'exécution, mémoire, similarité CFG (BinDiff)
- Pourquoi BCF et FLA ont significativement plus d'impact que SUB ou MBA au niveau binaire
- Comment intégrer un obfuscateur LLVM dans un système de build C multi-makefile existant
- La différence entre la force d'obfuscation et l'adéquation pour la production — Tigress produisait les binaires les plus obfusqués mais crashait sur du code réel
- Les contraintes de cross-compilation dans un contexte de sécurité embarquée RISC-V

---

## Ressources

- [Hikari Obfuscator](https://github.com/HikariObfuscator/Hikari)
- [O-LLVM](https://github.com/obfuscator-llvm/obfuscator)
- [Pluto Obfuscator](https://github.com/bluesadi/Pluto)
- [Tigress](https://tigress.wtf/introduction.html)
- [BinDiff](https://www.zynamics.com/bindiff.html)
- [Collberg et al. — A Taxonomy of Obfuscating Transformations (1997)](https://researchspace.auckland.ac.nz/handle/2292/3491)
