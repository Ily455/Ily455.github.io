---
title: "IM-LLVM-Pass"
date: 2024-07-01
draft: false
description: "Un pass compilateur LLVM qui mangling les noms de symboles internes au niveau IR via un PRNG seedé — complique significativement le reverse engineering sans changer la sémantique du programme."
tags: ["llvm", "compilateur", "obfuscation", "reverse-engineering", "cpp", "bas-niveau"]
---

## Ce que ça fait

IM-LLVM-Pass est un pass compilateur qui s'exécute dans le pipeline de compilation LLVM et renomme toutes les fonctions et variables globales internes (non exportées) en chaînes aléatoires — avant que le binaire soit produit.

```
Code source → Clang → LLVM IR → [IM-LLVM-Pass] → IR manglé → Code objet → Binaire
```

Résultat : un binaire où les symboles internes comme `checkLicenseKey` ou `decrypt_payload` deviennent `_Zf3a9b1c` — le programme se comporte identiquement, mais les ingénieurs en reverse engineering ne peuvent plus utiliser les noms de symboles comme point de départ.

---

## Pourquoi au niveau IR

Le pass opère sur **LLVM IR** (Intermediate Representation), pas sur le code source ni le binaire final.

C'est important parce que :
- **L'obfuscation au niveau source** nécessite de comprendre l'AST et la sémantique du langage — fragile, spécifique au langage
- **Le patching binaire** après compilation peut casser les relocations et les infos de debug
- **Au niveau IR** : indépendant du langage (tout langage qui compile vers LLVM fonctionne), opère avant la génération finale du code, et peut renommer les symboles en toute sécurité en préservant les cross-références

---

## Comment ça fonctionne

Le pass est un **Module Pass** — il voit le programme entier en une fois, pas juste une fonction.

```
Pour chaque fonction dans le module :
  → Skip si linkage externe (exportée — le renommage casserait l'ABI)
  → Skip si nommée "main" (le point d'entrée doit rester trouvable)
  → Générer un nouveau nom : seeder le PRNG avec le nom du symbole → produire une chaîne aléatoire
  → Renommer la fonction partout où elle est référencée

Idem pour les variables globales.
```

### Génération des noms

Le PRNG utilisé est `std::mt19937` (Mersenne Twister), seedé avec un hash du nom de symbole original. Le renommage est **déterministe** — même source, même pass, même sortie à chaque fois. Utile pour les builds reproductibles et le débogage du pass.

```cpp
std::mt19937 rng(std::hash<std::string>{}(originalName));
std::string mangledName = generateName(rng);
```

Les noms générés ressemblent à des symboles générés par le compilateur, ce qui les fait se fondre dans la masse plutôt que d'être reconnaissables comme obfusqués.

---

## Build & Utilisation

### Prérequis

- Headers de développement LLVM 14+
- CMake 3.13+
- Clang (pour compiler les cibles)

```bash
# Compiler le pass
mkdir build && cd build
cmake .. -DLLVM_DIR=/path/to/llvm/lib/cmake/llvm
make

# Compiler une cible avec le pass chargé
clang -fpass-plugin=./build/libManglePass.so -O1 target.c -o target_mangled
```

### Ce qui change

Avant le pass — `nm target | grep T` :
```
000000000000 T main
000000000000 T checkLicenseKey
000000000000 T decrypt_payload
000000000000 T computeChecksum
```

Après le pass :
```
000000000000 T main
000000000000 T _Zf3a9b1c
000000000000 T _Z8d2e4f1a
000000000000 T _Z1b7c9d3e
```

`main` est préservé. Tout le reste a disparu.

---

## Diff IR et Assembleur

Le pass produit des changements visibles au niveau IR :

**Avant (extrait test.ll) :**
```llvm
define internal i32 @addNumbers(i32 %a, i32 %b) {
  %result = add i32 %a, %b
  ret i32 %result
}

define i32 @main() {
  %r = call i32 @addNumbers(i32 3, i32 4)
  ret i32 %r
}
```

**Après (extrait mangled-test.ll) :**
```llvm
define internal i32 @_Zf3a9b1c(i32 %a, i32 %b) {
  %result = add i32 %a, %b
  ret i32 %result
}

define i32 @main() {
  %r = call i32 @_Zf3a9b1c(i32 3, i32 4)
  ret i32 %r
}
```

Le site d'appel dans `main` est mis à jour automatiquement — le pass gère toutes les références.

---

## Limites

Il s'agit d'un projet d'apprentissage — pas de l'obfuscation de production. Limites connues :

- **`main` est toujours préservé** — nécessaire pour que le binaire fonctionne, mais c'est un point d'entrée connu
- **Symboles externes intacts** — tout ce qui a un linkage externe garde son nom (par conception — le renommage casserait l'ABI)
- **Symboles de debug** — si compilé avec `-g`, les infos DWARF peuvent encore contenir les noms originaux
- **Pas d'obfuscation du flot de contrôle** — le renommage seul ne change pas le graphe de flot de contrôle, que la plupart des vrais reverse engineers analysent
- **Outil standalone** — non intégré dans un système de build ou une pipeline CI ; doit être chargé explicitement

Pour des besoins d'obfuscation réels, des outils comme [Hikari](https://github.com/HikariObfuscator/Hikari) ou [OLLVM](https://github.com/obfuscator-llvm/obfuscator) ajoutent l'aplatissement du flot de contrôle, le bogus control flow et la substitution d'instructions en plus du renommage.

---

## Ce que j'en retiens

Construire ce pass a nécessité de comprendre l'infrastructure de passes de LLVM à un niveau que les tutoriels génériques ne couvrent pas :

- La différence entre Function Passes et Module Passes — et pourquoi le renommage de symboles nécessite un scope module
- Comment LLVM suit les références de symboles — renommer une fonction nécessite de mettre à jour chaque `call` et `reference` dans l'IR, pas seulement la définition
- Pourquoi les symboles avec linkage `external` ne peuvent pas être renommés — ils font partie du contrat ABI avec le linker
- Comment fonctionne le seedage de `std::mt19937` et pourquoi l'obfuscation déterministe compte pour la reproductibilité

---

## Ressources

- [Documentation LLVM Writing a Pass](https://llvm.org/docs/WritingAnLLVMPass.html)
- [LLVM Language Reference (IR)](https://llvm.org/docs/LangRef.html)
- [LLVM Programmer's Manual](https://llvm.org/docs/ProgrammersManual.html)
- [Hikari — Obfuscateur LLVM](https://github.com/HikariObfuscator/Hikari)
