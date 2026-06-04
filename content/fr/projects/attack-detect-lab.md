---
title: "ATT&CK Detection Lab"
date: 2026-05-31
draft: false
description: "Lab local pour simuler des techniques MITRE ATT&CK et les détecter avec Elastic SIEM — Atomic Red Team contre une cible Linux, logs envoyés dans Kibana."
tags: ["elastic", "siem", "docker", "mitre-attack", "atomic-red-team", "detection", "blue-team", "linux", "auditd"]
---

> Projet personnel, 2026. Un environnement autonome pour simuler des techniques adversariales et les détecter avec Elastic SIEM — 6 techniques MITRE ATT&CK, preuves log réelles et règles de détection Kibana.

---

## Vue d'ensemble

Le lab exécute des techniques Atomic Red Team contre une cible Linux dédiée et les détecte avec Elastic SIEM. Le stack SIEM tourne dans Docker sur un Mac Apple Silicon. Les techniques s'exécutent sur une VM Linux séparée (UTM), qui envoie ses logs vers Elasticsearch via Filebeat.

Chaque technique produit : des logs bruts depuis auditd sur la cible Linux, une règle Sigma, une requête KQL validée dans Kibana Security, et un writeup documentant les preuves et la logique de détection.

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  macOS Host (M3)                                             │
│                                                              │
│  run-test.sh                                                 │
│    └── SSH → VM Linux (UTM)                                  │
│               ├── Atomic Red Team exécute les techniques     │
│               ├── auditd enregistre chaque syscall execve    │
│               └── Filebeat ──► Elasticsearch ◄── Kibana      │
│                                (Docker)         (Docker)     │
└──────────────────────────────────────────────────────────────┘
```

Atomic Red Team et toute la simulation d'attaque restent sur la cible Linux. Le Mac ne fait qu'initier les tests via SSH. Le SIEM reçoit les logs mais ne touche jamais la cible directement.

---

## Stack

| Composant | Rôle | Version |
|---|---|---|
| Elasticsearch | Stockage et indexation des logs | 8.14.3 |
| Kibana | Interface SIEM, règles de détection, KQL | 8.14.3 |
| Filebeat | Collecteur de logs (sur la VM cible) | 8.14.3 |
| VM cible | Ubuntu 22.04 ARM64, surface d'attaque | UTM sur M3 |
| Atomic Red Team | Simulateur de techniques MITRE ATT&CK | latest |
| PowerShell | Runtime Atomic | 7.4.6 |

Les images Elastic tournent en `linux/arm64` — exécution native sur M3, sans émulation.

---

## Déroulement d'un test

```bash
./run-test.sh T1059.004
```

1. SSH vers la VM Linux cible
2. Exécution d'`Invoke-AtomicTest T1059.004` sur la cible
3. auditd enregistre chaque appel `execve` — chaque commande exécutée, avec ses arguments
4. Filebeat envoie les logs auditd vers Elasticsearch en quelques secondes
5. Les événements apparaissent dans Kibana sous la data view `attack-detect-logs-*`
6. `Invoke-AtomicTest T1059.004 -Cleanup` supprime les artefacts

---

## Techniques

| # | ID | Nom | Tests | Tactique |
|---|---|---|---|---|
| 1 | T1059.004 | Unix Shell | 15/17 | Execution |
| 2 | T1053.003 | Persistance via Cron | 4/4 | Persistence |
| 3 | T1136.001 | Création de compte local | 2/5 | Persistence |
| 4 | T1087.001 | Découverte de comptes locaux | 4/4 | Discovery |
| 5 | T1083 | Découverte de fichiers et répertoires | 3/3 | Discovery |
| 6 | T1105 | Transfert d'outils | 7/9 | Command & Control |

### T1059.004 — Unix Shell

15/17 sous-tests réussis. auditd a capturé chaque invocation de shell via `execve`. Extrait :
```
type=EXECVE msg=audit(1780489845.066:1912): argc=3 a0="sh" a1="-c" a2=""
```
**KQL :** `message: "type=EXECVE" AND (message: "a0=\"sh\"" OR message: "a0=\"bash\"" OR message: "a0=\"dash\"")`

### T1053.003 — Persistance via Cron

4/4 sous-tests réussis. Preuve clé — crontab chargé depuis `/tmp/`, ce que la gestion légitime de cron ne fait jamais :
```
type=EXECVE argc=2 a0="crontab" a1="/tmp/persistevil"
```
**KQL :** `message: "type=EXECVE" AND message: "a0=\"crontab\""`

### T1136.001 — Création de compte local

2/5 tests Linux réussis (tests FreeBSD et kubectl exclus). Inclut la création d'un compte UID=0 — un second root. `auid=1000` persiste après sudo, traçant l'utilisateur d'origine même quand les commandes tournent en root.
```
type=PROCTITLE proctitle=7573657264656C006576696C5F75736572  →  userdel evil_user
```
**KQL :** `message: "type=EXECVE" AND (message: "a0=\"useradd\"" OR message: "a0=\"adduser\"")`

### T1087.001 — Découverte de comptes locaux

4/4 tests réussis. Commandes : `cat /etc/passwd`, `cat /etc/shadow`, `lsof`, `getent passwd`. auditd a capturé chaque invocation.
```
type=EXECVE argc=2 a0="cat" a1="/etc/passwd"
type=EXECVE argc=1 a0="lsof"
```
**KQL :** `message: "type=EXECVE" AND (message: "a1=\"/etc/passwd\"" OR message: "a1=\"/etc/shadow\"" OR message: "a0=\"lsof\"")`

### T1083 — Découverte de fichiers et répertoires

3/3 tests réussis. Commandes : `find /`, énumération récursive de répertoires, `showmount` pour la découverte de partages réseau.
```
type=EXECVE argc=2 a0="find" a1="/"
type=EXECVE argc=1 a0="showmount"
```
**KQL :** `message: "type=EXECVE" AND (message: "a0=\"find\"" OR message: "a0=\"showmount\"" OR message: "a0=\"tree\"")`

### T1105 — Transfert d'outils

7/9 tests réussis. Un conteneur `remote` dédié (Ubuntu + SSH + rsync) a été ajouté comme serveur de staging. rsync push/pull, scp push/pull, sftp push, et curl download+execute ont tous réussi.
```
type=EXECVE a0="curl" a1="-s" a2="<url>"
```
**KQL :** `message: "type=EXECVE" AND (message: "a0=\"curl\"" OR message: "a0=\"wget\"" OR message: "a0=\"rsync\"" OR message: "a0=\"scp\"")`

---

## Contraintes techniques découvertes pendant la construction

La première version de ce lab utilisait un conteneur Docker comme cible. Cela a révélé plusieurs contraintes importantes.

**Le noyau linuxkit de Docker Desktop n'a pas de sous-système audit.** Le framework audit (`auditd`, `CAP_AUDIT_*`) n'est pas compilé dans le noyau que Docker Desktop embarque. `auditctl -s` retourne "Operation not permitted" quelle que soit la configuration des capabilities. Aucun contournement sans remplacer le noyau.

**Le script block logging PowerShell sous Linux nécessite systemd/journald.** Activer `ScriptBlockLogging` dans `powershell.config.json` ne fait rien silencieusement dans un conteneur minimal — il n'y a pas de journald pour recevoir les logs.

**Conséquence :** Une cible en conteneur ne peut capturer que ce qu'elle écrit délibérément dans des fichiers connus. Elle ne peut pas enregistrer passivement quelles commandes ont été exécutées. Ce n'est pas de la surveillance SIEM réelle.

**Solution :** Déplacer la cible vers une vraie VM Linux (UTM, Ubuntu 22.04). Deux règles audit capturent chaque appel `execve` sur le système :
```bash
auditctl -a always,exit -F arch=b64 -S execve -k exec_commands
auditctl -a always,exit -F arch=b32 -S execve -k exec_commands
```

**Note ARM64 :** Le dépôt de packages Linux de Microsoft ne publie pas PowerShell pour ARM64. L'installation nécessite l'archive directement depuis les releases GitHub.
