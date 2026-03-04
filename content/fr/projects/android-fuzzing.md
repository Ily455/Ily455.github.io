---
title: "Étude de fuzzing Android"
date: 2023-05-01
draft: false
tags: ["fuzzing", "android", "research", "security"]
description: "Étude de 3 mois sur le fuzzing Android : approches générative et mutationnelle, AFL, environnement à deux VM."
---

Étude et mise en pratique du fuzzing sur Android, réalisée comme projet académique sur environ 3 mois.

## Ce que j’ai fait

- Étudié les techniques de fuzzing génératives et mutationnelles
- Évalué plusieurs fuzzers open source, notamment AFL et Droid-FF
- Mis en place un environnement à deux VM (Linux + Android via ADB sur TCP) pour tenter de reproduire une vulnérabilité multimédia Android connue (2014)
- Généré de grands volumes d’échantillons multimédias malformés et les injecter dans une build Android vulnérable

## Ce qu’il s’est réellement passé

Aucune vulnérabilité n’a été reproduite. En cours de projet, j’ai regardé une présentation d’une entreprise faisant de la recherche fuzzing Android à un niveau professionnel : ils expliquaient que leur setup reposait sur des pools d’appareils physiques, car les environnements émulés introduisent trop de variables et masquent des crashs réels. Cela a remis en perspective mon setup à deux VM.

Le projet n’a pas donné de résultat exploitable côté vulnérabilité, mais il m’a apporté une meilleure compréhension de la difficulté réelle de ce type de recherche, de l’infrastructure nécessaire et des limites pratiques.

## Ce que j’ai appris

- Les bases du fuzzing : génération vs mutation
- ADB, internals Android et limites de l’émulation
- L’écart entre un setup académique et une infra de recherche en production
- Comment présenter proprement des résultats négatifs
