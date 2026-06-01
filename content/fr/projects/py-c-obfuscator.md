---
title: "Py-C-obfuscator"
date: 2024-06-01
draft: false
tags: ["obfuscation", "python", "c", "compiler"]
description: "Un obfuscateur C source-à-source écrit en Python."
github: "https://github.com/Ily455/Py-C-obfuscator"
---

Un obfuscateur C source-à-source écrit en Python. Il prend du code C en entrée et produit une version fonctionnellement équivalente mais obfusquée.

## Ce que le projet fait

Applique des transformations d’obfuscation au niveau du code source avant compilation, afin de rendre le code plus difficile à lire et à analyser sans en changer le comportement.

![Pipeline de transformation source-à-source](/images/projects/py-c-obfuscator/pipeline.svg)

## Pourquoi je l’ai construit

Dans le cadre de mes travaux sur l’obfuscation chez Secure-IC, je voulais explorer l’obfuscation au niveau source en complément des approches au niveau compilateur. Python était un bon choix pour prototyper rapidement des transformations AST.

## Ce que j’ai appris

- Les compromis entre obfuscation au niveau source et au niveau IR
- Le parsing et la manipulation d’AST C en Python
- Ce qui survit (ou non) aux optimisations du compilateur
