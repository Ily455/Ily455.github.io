---
title: "ATT&CK Detection Lab"
date: 2026-05-31
draft: false
description: "Lab local pour simuler des techniques MITRE ATT&CK et les détecter avec Elastic SIEM — tout tourne sous Docker, pas de VMs, nettoyage en une commande."
tags: ["elastic", "siem", "docker", "mitre-attack", "atomic-red-team", "detection", "blue-team", "sigma", "linux"]
---

> Projet personnel, 2026. Objectif : construire un environnement autonome pour simuler des techniques adversariales et concevoir les détections — une technique à la fois, avec des logs, des règles Sigma et des writeups en sortie.

---

## Vue d'ensemble

Le lab exécute des techniques Atomic Red Team contre un conteneur Linux cible et les détecte avec Elastic SIEM. Le pipeline complet — simulation d'attaque, collecte de logs, détection — tourne en local sur un Mac Apple Silicon, sans VMs ni dépendance cloud.

Chaque technique produit :
- Des logs bruts capturés depuis le système de logging unifié macOS
- Une règle Sigma de détection
- Une requête KQL pour Kibana
- Un writeup documentant la technique, les preuves et la logique de détection

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  macOS Host (M3)                                            │
│                                                             │
│  run-test.sh                                                │
│    └── pwsh → Invoke-AtomicTest ──► SSH → conteneur cible   │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Docker  (réseau bridge : elastic)                   │   │
│  │                                                      │   │
│  │  target ──► Filebeat ──► Elasticsearch ◄── Kibana   │   │
│  │  (Ubuntu)   surveille     stocke +         SIEM UI  │   │
│  │             /logs/        indexe                     │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Décision de conception clé :** Atomic Red Team s'exécute dans le conteneur cible via SSH — pas sur le Mac hôte. Le Mac ne fait qu'initier le test. PowerShell et le framework Atomic ne laissent aucune trace sur l'hôte après nettoyage, au-delà de Docker Desktop lui-même.

---

## Stack

| Composant | Rôle | Version |
|---|---|---|
| Elasticsearch | Stockage et indexation des logs | 8.14.3 |
| Kibana | Interface SIEM, règles de détection, KQL | 8.14.3 |
| Filebeat | Collecteur de logs | 8.14.3 |
| Conteneur cible | Ubuntu ARM64, surface d'attaque | — |
| Atomic Red Team | Simulateur de techniques MITRE ATT&CK | latest |
| PowerShell | Runtime Atomic (transport SSH) | latest |

Toutes les images Elastic tournent en `linux/arm64` — exécution native sur M3, sans émulation.

---

## Déroulement d'un test

```bash
./run-test.sh T1059.004
```

1. Ouvre une session SSH depuis PowerShell vers le conteneur cible
2. Exécute `Invoke-AtomicTest T1059.004` dans le conteneur
3. Attend que les événements post-exécution se stabilisent
4. Lance `Invoke-AtomicTest T1059.004 -Cleanup` pour supprimer les artefacts
5. Filebeat envoie les logs du conteneur vers Elasticsearch en quelques secondes
6. Les événements apparaissent dans Kibana sous la data view `attack-detect-logs-*`

---

## Techniques

| # | ID | Nom | Statut |
|---|---|---|---|
| 1 | T1059.004 | Unix Shell | Planifié |
| 2 | T1053.003 | Persistance via Cron | Planifié |
| 3 | T1136.001 | Création de compte local | Planifié |
| 4 | T1087.001 | Découverte de comptes locaux | Planifié |
| 5 | T1083 | Découverte de fichiers et répertoires | Planifié |
| 6 | T1105 | Transfert d'outils | Planifié |

---

## Pourquoi Docker plutôt que des VMs

Les VMs ajoutent une couche coûteuse et lente à itérer. Docker offre :
- Stack complète opérationnelle en moins de 2 minutes
- `docker compose down -v` efface tout — aucun état résiduel entre les tests
- Images ARM64 natives sur M3, sans overhead Rosetta
- Le conteneur cible est un vrai environnement Linux — les techniques se comportent comme sur un vrai hôte Linux

La contrepartie : le système de logging unifié macOS est propriétaire et moins structuré qu'auditd sous Linux ou l'Event Log Windows. Le lab est conçu avec une architecture v2 en tête qui déplace tout dans des conteneurs pour un vrai setup Linux-vers-Linux.

---

## Roadmap — v2

Déplacer Atomic Red Team dans un conteneur attaquant dédié. Mouvement latéral SSH entre conteneurs, sans implication de l'hôte. Le Mac reste complètement propre.

```
Mac (client SSH uniquement)
  └── SSH → Conteneur attaquant (PowerShell + Atomic)
                └── SSH → Conteneur cible (Ubuntu + Filebeat)
                              └── Filebeat → Elasticsearch → Kibana
```
