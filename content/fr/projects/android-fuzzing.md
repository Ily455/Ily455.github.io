---
title: "Fuzzing Android — Retours pratiques après des résultats négatifs"
date: 2026-03-06
draft: false
description: "Étude de 3 mois sur le fuzzing de systèmes Android avec AFL et Droid-FF. Mise en place de l'infrastructure, tentative de reproduction d'une vulnérabilité, et compte rendu honnête de ce qui n'a pas fonctionné."
tags: ["fuzzing", "android", "afl", "recherche-securite", "adb"]
---

> Projet de recherche — janvier à mai 2023. Objectif : étudier les techniques de fuzzing génératif et mutatif, évaluer des fuzzers Android open source, monter un environnement réel et tenter de reproduire une vulnérabilité connue dans la couche multimédia Android. On n'y est pas totalement arrivé — et documenter pourquoi, c'est l'essentiel.

---

## Objectif

Deux volets :

1. **Étude** — analyser les techniques de fuzzing et évaluer les outils existants pour Android : AFL, Droid-FF, et ce qui existait à l'époque
2. **Pratique** — construire un environnement fonctionnel et tenter de reproduire une vulnérabilité connue dans la couche multimédia Android (Stagefright et ses successeurs)

Cible visée : le parsing de fichiers média. La couche multimédia Android a historiquement été une surface d'attaque riche — des entrées MP4, H.264 ou AAC malformées envoyées au décodeur ont produit de vrais CVEs.

---

## Contexte — Pourquoi le fuzzing Android est difficile

Fuzzer un binaire desktop est relativement simple : compiler avec AddressSanitizer, pointer un fuzzer sur le binaire, attendre les crashs. Android ajoute des frictions à chaque niveau :

```
Fuzzing desktop :
  Fuzzer → Binaire cible → Crash → Terminé

Fuzzing Android :
  Fuzzer → Pont ADB → VM/device Android
         → App sandboxée → IPC → Processus système (mediaserver)
         → HAL → (comportement spécifique au matériel)
         → Crash quelque part → (peut-être) détecté → renvoyé via ADB
```

Chaque flèche est un point de défaillance potentiel. Le pont ADB seul introduit assez de latence pour détruire le débit de fuzzing. Récupérer le feedback de couverture depuis un processus distant sandboxé n'est pas trivial.

---

## Outils évalués

### AFL — American Fuzzy Lop

AFL est le fuzzer à guidage par couverture de référence. Il instrumente un binaire cible à la compilation pour tracer les chemins d'exécution. Les entrées qui atteignent de nouveaux chemins sont conservées et mutées — c'est ce qui le rend "intelligent" par rapport à une mutation purement aléatoire.

```
Corpus → Mutation → Exécution → Bitmap couverture → Nouveau chemin ? → Garder l'entrée
                                                   ↓
                                              Muter à nouveau
```

AFL fonctionne bien quand :
- On contrôle la compilation (instrumentation AFL + `-fsanitize=address`)
- La cible lit depuis stdin ou un fichier
- L'exécution est rapide (milliers d'itérations par seconde)

Limites d'AFL pour Android :
- Les binaires système ne peuvent pas facilement être recompilés avec l'instrumentation AFL
- Le mode QEMU (fuzzing boîte noire sans instrumentation) fonctionne mais est 2 à 5 fois plus lent
- ADB ajoute une latence aller-retour — on ne peut pas approcher les débits nécessaires pour qu'AFL soit efficace

### Droid-FF

Droid-FF est un framework de fuzzing spécifique à Android, conçu pour fonctionner via ADB. Il gère la couche de communication pour qu'on puisse se concentrer sur la logique de fuzzing.

Évalué face à AFL sur :
- Complexité de mise en place
- Débit (itérations/seconde)
- Qualité du feedback de couverture
- Intégration avec des corpus existants

En pratique : Droid-FF a simplifié la communication ADB mais n'a pas résolu le problème fondamental de latence. Les deux outils ont fini bottleneckés par le pont.

---

## Architecture de l'environnement

### Deux VMs en réseau

```
┌─────────────────────────────────┐
│  Hôte (Linux)                   │
│                                 │
│  ┌───────────────┐              │
│  │ VM Fuzzer     │              │
│  │ (Ubuntu)      │──── ADB/TCP ─┼──┐
│  │ AFL / Droid-FF│              │  │
│  └───────────────┘              │  │
│                                 │  │
│  ┌───────────────┐              │  │
│  │ VM Android    │◄─────────────┘  │
│  │ (QEMU/AVD)    │                 │
│  │ Processus cible│                │
│  └───────────────┘                 │
└─────────────────────────────────┘
```

ADB sur TCP entre les deux VMs — pas de USB. Cela s'avère avoir peu d'impact sur le débit, les autres goulots d'étranglement étant bien plus importants.

### Cible

Le plan était de cibler `stagefright` / `mediaserver` — le composant Android responsable du parsing et du décodage des fichiers média. Envoyer des fichiers MP4 malformés pour déclencher des bugs de parsing.

```bash
# Pousser un fichier de test vers le device
adb push malformed.mp4 /sdcard/test.mp4

# Déclencher le parsing via intent
adb shell am start -a android.intent.action.VIEW \
  -d file:///sdcard/test.mp4 -t video/mp4
```

---

## Ce qui n'a pas fonctionné et pourquoi

### 1. Débit trop faible

AFL a besoin de milliers d'exécutions par seconde pour être efficace. Avec le pont ADB, on obtenait 5 à 20 exécutions par seconde. Ce n'est pas suffisant pour couvrir l'espace de recherche de façon significative.

Solutions testées :
- ADB sur TCP (légère amélioration par rapport à USB)
- Réduction de la taille des cas de test
- Parallélisation sur plusieurs VMs Android

Même avec la parallélisation, le débit restait bien en dessous du seuil d'efficacité.

### 2. Détection des crashs peu fiable

Quand `mediaserver` crashe sur Android, le système le redémarre automatiquement. Détecter si un crash a eu lieu — et quelle entrée l'a provoqué — nécessitait de parser la sortie `logcat`, ce qui ajoutait latence et complexité.

```bash
adb logcat -s "AndroidRuntime:E" "DEBUG:*"
```

Ça fonctionnait mais c'était fragile. Les crashs de processus système non liés apparaissaient dans le même flux de logs.

### 3. L'émulateur se comportait différemment du matériel réel

Certains comportements spécifiques aux implémentations HAL Qualcomm ou MediaTek ne pouvaient pas être reproduits dans l'émulateur Android basé sur QEMU. La vulnérabilité qu'on cherchait à reproduire avait des composantes spécifiques au matériel.

### 4. Qualité du corpus

Un bon corpus de fuzzing nécessite des fichiers seeds valides qui exercent les chemins de code visés. On a démarré avec des fichiers MP4 génériques, mais sans savoir exactement quels chemins de codec étaient vulnérables, la sélection des seeds était essentiellement un tâtonnement.

---

## Ce qui a fonctionné

Même sans reproduire la vulnérabilité cible, le projet a produit des résultats utilisables :

**Infrastructure :** Un environnement de fuzzing deux VMs fonctionnel via ADB/TCP, capable de pousser des fichiers et déclencher des intents de parsing média de façon fiable. Réutilisable pour de futures cibles.

**Triage de crashs :** Un script de parsing logcat qui filtre les crashs et les corrèle avec le fichier d'entrée qui les a déclenchés.

**Corpus :** Un ensemble de fichiers MP4 mutés, dont certains ont causé des anomalies non-crashantes (ANR, erreurs de parser) qui méritaient investigation.

**Compréhension :** Une vision claire des goulots d'étranglement du fuzzing Android et de ce qu'il faudrait pour les surmonter — principalement, obtenir de l'instrumentation à l'intérieur de `mediaserver` pour récupérer un vrai feedback de couverture.

---

## Ce qu'il faudrait pour le faire vraiment

1. **Fuzzing in-process** — compiler un harness autonome qui appelle directement le parseur média, sans le surcoût ADB. C'est ce que fait l'[intégration libFuzzer spécifique Android](https://android.googlesource.com/platform/external/libFuzzer/).

2. **Meilleur corpus** — utiliser une mutation consciente du format (connaître la structure des boîtes MP4) plutôt qu'une mutation pure au niveau octet. Des fuzzers comme Atheris ou des fuzzers basés sur une grammaire seraient plus efficaces.

3. **Matériel réel avec ASAN** — compiler Android depuis les sources avec AddressSanitizer pour obtenir la détection d'erreurs mémoire sur du matériel réel.

---

## Ce que j'en retiens

Le bilan le plus honnête : les résultats négatifs sont des résultats. Les limites d'infrastructure qu'on a rencontrées sont réelles — ce n'est pas propre à ce lab, c'est pourquoi le fuzzing Android reste un sujet de recherche actif.

Plus concrètement :
- Le fuzzing à guidage par couverture requiert un débit élevé — tout ce qui ajoute de la latence (ADB, VMs, IPC) tue l'efficacité
- La détection de crashs sur un OS live est plus difficile que sur un binaire seul — il faut filtrer le bruit
- La fidélité de l'émulateur compte pour les vulnérabilités spécifiques au matériel
- La mutation consciente du format est significativement plus efficace que le bit-flipping pour des formats structurés

---

## Ressources

- [Documentation AFL](https://github.com/google/AFL)
- [Droid-FF](https://github.com/interference-security/droid-ff)
- [libFuzzer sur Android](https://android.googlesource.com/platform/external/libFuzzer/)
- [Stagefright — Joshua Drake, Black Hat 2015](https://www.blackhat.com/docs/us-15/materials/us-15-Drake-Stagefright-Scary-Code-In-The-Heart-Of-Android.pdf)
- [Bulletins de sécurité Android](https://source.android.com/docs/security/bulletin)