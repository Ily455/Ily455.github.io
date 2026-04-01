---
title: "DroidTest"
date: 2026-03-06
draft: false
tags: ["python", "android", "adb", "cli", "testing", "automation"]
description: "Un outil CLI Python qui exécute des lots de commandes ADB de diagnostic sur des appareils Android, rapporte les résultats commande par commande et exporte en JSON."
github: "https://github.com/Ily455/DroidTest"
---

Un runner en ligne de commande pour les diagnostics d'appareils Android. On lui donne une liste de sous-commandes `adb`, il les exécute toutes et produit un rapport pass/fail structuré.

## Ce que le projet fait

Automatise la boucle de test : définir la liste une fois, lancer, obtenir un rapport. Utile pour vérifier un appareil après un flash, faire des contrôles QA matériel (capteurs, réseau, stockage) ou obtenir un snapshot rapide de l'état d'un device connecté.

## Architecture

Le projet a été refactorisé d'un script monolithique vers un package structuré :

```
DroidTest.py          ← point d'entrée (rétrocompatible)
droidtest/
  core.py             ← chargement des commandes et logique d'exécution
  cli.py              ← parsing des arguments, affichage, bannière
list.txt              ← liste de commandes par défaut
tests/
  test_core.py        ← tests unitaires
```

## Ce que j'ai appris

- Séparation des responsabilités : logique métier vs CLI vs configuration
- Gestion robuste des sous-processus avec `shlex.split` et timeouts
- Codes de sortie structurés : `0` tout passe, `2` au moins un échec, `1` aucune commande trouvée
