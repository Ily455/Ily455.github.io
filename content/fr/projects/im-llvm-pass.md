---
title: "IM-LLVM-Pass"
date: 2024-07-01
draft: false
tags: ["llvm", "obfuscation", "compiler", "security"]
description: "Un pass LLVM pour le renommage d’identifiants : fonctions, variables globales et structures."
github: "https://github.com/Ily455/IM-LLVM-Pass"
---

Un pass compilateur LLVM qui effectue du renommage d’identifiants : noms de fonctions, variables globales et structures sont randomisés à la compilation.

## Ce que le projet fait

Le pass s’intègre dans la chaîne de compilation LLVM et renomme les identifiants via un schéma aléatoire. L’objectif est de compliquer le reverse engineering en supprimant les noms symboliques parlants dans le binaire final.

## Pourquoi je l’ai construit

Ce projet vient de mon stage chez Secure-IC, où j’étudiais des techniques d’obfuscation et leur impact sur l’analyse binaire. Après avoir utilisé des outils LLVM existants, j’ai voulu implémenter mon propre pass pour mieux comprendre les internals de la chaîne de compilation.

## Ce que j’ai appris

- Le fonctionnement des passes LLVM et leur intégration dans la compilation
- Les différences entre obfuscation niveau source, IR et binaire
- Les limites du renommage d’identifiants pris isolément
