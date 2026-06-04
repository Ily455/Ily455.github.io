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

Chaque technique complétée produit :
- Des logs bruts depuis auditd et syslog sur la cible Linux
- Une règle Sigma de détection
- Une requête KQL pour Kibana
- Un writeup documentant la technique, les preuves et la logique de détection

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  macOS Host (M3)                                             │
│                                                              │
│  run-test.sh                                                 │
│    └── SSH → VM Linux (UTM)                                  │
│               ├── Atomic Red Team exécute les techniques     │
│               ├── auditd enregistre chaque syscall execve     │
│               └── Filebeat ──► Elasticsearch ◄── Kibana      │
│                                (Docker)         (Docker)     │
└──────────────────────────────────────────────────────────────┘
```

**Décision de conception clé :** Atomic Red Team et toute la simulation d'attaque restent sur la cible Linux — pas sur le Mac. Le Mac ne fait qu'initier les tests via SSH. Le SIEM (Elasticsearch + Kibana) tourne dans Docker et ne touche jamais la cible directement — il reçoit uniquement les logs.

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
4. Filebeat envoie les logs auditd et syslog vers Elasticsearch en quelques secondes
5. Les événements apparaissent dans Kibana sous la data view `attack-detect-logs-*`
6. `Invoke-AtomicTest T1059.004 -Cleanup` supprime les artefacts

---

## Techniques

| # | ID | Nom | Statut |
|---|---|---|---|
| 1 | T1059.004 | Unix Shell | Terminé |
| 2 | T1053.003 | Persistance via Cron | Terminé |
| 3 | T1136.001 | Création de compte local | Terminé |
| 4 | T1087.001 | Découverte de comptes locaux | Terminé |
| 5 | T1083 | Découverte de fichiers et répertoires | Terminé |
| 6 | T1105 | Transfert d'outils | Terminé |

### T1059.004 — Unix Shell

15/17 sous-tests exécutés avec succès. auditd a capturé chaque invocation de shell via `execve`.

Extrait de log :
```
type=EXECVE msg=audit(1780489845.066:1912): argc=3 a0="sh" a1="-c" a2=""
```

**Requête KQL :**
```
message: "type=EXECVE" AND (message: "a0=\"sh\"" OR message: "a0=\"bash\"" OR message: "a0=\"dash\"")
```

---

### T1053.003 — Persistance via Cron

4/4 sous-tests exécutés avec succès. auditd a capturé les écritures dans les répertoires cron et les commandes bash via les enregistrements PROCTITLE.

Preuve clé — crontab chargé depuis /tmp/ :
```
type=EXECVE argc=2 a0="crontab" a1="/tmp/persistevil"
```

**Requête KQL :**
```
message: "type=EXECVE" AND message: "a0=\"crontab\""
```

---

### T1136.001 — Création de compte local

Atomic a créé un utilisateur local nommé `evil_user`. auditd a capturé la création et la suppression via `useradd`/`userdel`.

Preuve clé — PROCTITLE décodé :
```
userdel evil_user
```

À noter : `auid=1000` persiste après sudo — auditd trace l'utilisateur d'origine (`ubuntu`) même quand la commande tourne en root.

**Requête KQL :**
```
message: "type=EXECVE" AND (message: "a0=\"useradd\"" OR message: "a0=\"adduser\"")
```

---

### T1087.001 — Découverte de comptes locaux

Atomic a exécuté des commandes d'énumération : `cat /etc/passwd`, `cat /etc/shadow`, `lsof`, `getent passwd`. auditd a capturé chaque invocation.

```
type=EXECVE argc=2 a0="cat" a1="/etc/passwd"
type=EXECVE argc=1 a0="lsof"
```

**Requête KQL :**
```
message: "type=EXECVE" AND (message: "a1=\"/etc/passwd\"" OR message: "a1=\"/etc/shadow\"" OR message: "a0=\"lsof\"")
```

---

### T1083 — Découverte de fichiers et répertoires

3/3 tests réussis : découverte de fichiers via `ls /tmp`, énumération récursive de répertoires, et `showmount` pour la découverte de partages réseau.

```
type=EXECVE argc=2 a0="find" a1="/"
type=EXECVE argc=1 a0="showmount"
```

**Requête KQL :**
```
message: "type=EXECVE" AND (message: "a0=\"find\"" OR message: "a0=\"showmount\"" OR message: "a0=\"tree\"")
```

---

### T1105 — Transfert d'outils

7/9 tests réussis. Un conteneur `remote` dédié (Ubuntu + SSH + rsync) a été ajouté pour permettre les tests de transfert. rsync push/pull, scp push/pull, sftp push/pull, et curl download+execute ont tous réussi.

```
type=EXECVE a0="curl" a1="-s" a2="<url>"
```

**Requête KQL :**
```
message: "type=EXECVE" AND (message: "a0=\"curl\"" OR message: "a0=\"wget\"")
```

---

## Contraintes techniques découvertes pendant la construction

La première version de ce lab utilisait un conteneur Docker comme cible plutôt qu'une VM. Cela a révélé plusieurs contraintes importantes.

**Le noyau linuxkit de Docker Desktop n'a pas de sous-système audit.** Faire tourner Ubuntu dans un conteneur Docker sur macOS signifie que le conteneur partage la VM Linux interne de Docker Desktop — pas un noyau standard. Le sous-système audit (`auditd`, `CAP_AUDIT_*`) n'est pas compilé dedans. `auditctl -s` retourne "Operation not permitted" quelle que soit la configuration des capabilities. Aucun contournement possible sans remplacer le noyau.

**Le script block logging PowerShell sous Linux nécessite systemd/journald.** Activer `ScriptBlockLogging` dans `powershell.config.json` ne fait rien silencieusement dans un conteneur minimal — il n'y a pas de journald pour recevoir les logs. Ils disparaissent.

**Conséquence :** Une cible en conteneur ne peut capturer que ce qu'elle écrit délibérément dans des fichiers de log connus — événements auth, cron, redirections explicites. Elle ne peut pas enregistrer passivement quelles commandes ont été exécutées. Ce n'est pas de la surveillance SIEM réelle.

**Solution :** Déplacer la cible vers une vraie VM Linux (UTM, Ubuntu 22.04). La VM dispose d'un vrai noyau, d'un vrai systemd, d'un vrai auditd. Deux règles audit capturent chaque appel `execve` sur le système — chaque commande exécutée, quelle que soit la façon dont l'attaquant l'a déclenchée.

```bash
auditctl -a always,exit -F arch=b64 -S execve -k exec_commands
auditctl -a always,exit -F arch=b32 -S execve -k exec_commands
```

**Note ARM64 :** Le dépôt de packages Linux de Microsoft ne publie pas PowerShell pour ARM64. L'installation nécessite de télécharger l'archive directement depuis les releases GitHub.

---

## Roadmap

- ~~Finaliser la VM UTM avec auditd + Filebeat~~ ✓
- ~~T1059.004 — Unix Shell~~ ✓
- ~~T1053.003 — Persistance via Cron~~ ✓
- ~~T1136.001 — Création de compte local~~ ✓
- ~~T1087.001 — Découverte de comptes locaux~~ ✓
- ~~T1083 — Découverte de fichiers et répertoires~~ ✓
- ~~T1105 — Transfert d'outils~~ ✓
