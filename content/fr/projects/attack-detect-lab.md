---
title: "ATT&CK Detection Lab"
date: 2026-05-31
draft: false
description: "Lab local pour simuler des techniques MITRE ATT&CK et les détecter avec Elastic SIEM — Atomic Red Team contre une cible Linux, logs envoyés dans Kibana."
tags: ["elastic", "siem", "docker", "mitre-attack", "atomic-red-team", "detection", "blue-team", "linux", "auditd"]
---

**[→ GitHub : Ily455/attack-detect-lab](https://github.com/Ily455/attack-detect-lab)**

> Projet personnel, 2026. Simulation de techniques adversariales avec Atomic Red Team contre une VM Linux, détectées avec Elastic SIEM — 6 techniques MITRE ATT&CK, preuves log auditd réelles et règles de détection Kibana.

---

## Architecture

![ATT&CK Detection Lab — architecture complète](/images/projects/attack-detect-lab/architecture.svg)

Le Mac joue le rôle de l'**attaquant** — il héberge Atomic Red Team et envoie les commandes à la VM via PSRemoting (transport SSH PowerShell). La VM est la **cible** — elle exécute les techniques, enregistre chaque syscall `execve` via auditd, et envoie les logs au SIEM via Filebeat. Le SIEM (Elasticsearch + Kibana) tourne dans Docker et ne touche jamais la cible.

---

## Stack

| Composant | Rôle | Version |
|---|---|---|
| Elasticsearch | Stockage et indexation des logs | 8.14.3 |
| Kibana | Interface SIEM, règles de détection, KQL | 8.14.3 |
| Filebeat | Collecteur de logs (sur la VM cible) | 8.14.3 |
| VM cible | Ubuntu 22.04 ARM64, surface d'attaque | UTM sur M3 |
| Atomic Red Team | Simulateur de techniques MITRE ATT&CK | latest |
| PowerShell | Transport PSRemoting + runtime Atomic | 7.4.6 |

Les images Elastic tournent en `linux/arm64` — sans émulation.

---

## Pipeline de détection

![Pipeline de détection — de l'exécution à l'alerte](/images/projects/attack-detect-lab/detection-flow.svg)

---

## Déroulement d'un test

```bash
./run-test.sh T1059.004
```

1. `pwsh` ouvre une PSSession (transport SSH) vers la VM en root
2. `Invoke-AtomicTest T1059.004 -Session $s` tourne sur le Mac, les commandes s'exécutent sur la VM
3. Le noyau Linux intercepte chaque syscall `execve` — auditd écrit un enregistrement `EXECVE` structuré
4. Filebeat lit `/var/log/audit/audit.log` et envoie les événements vers Elasticsearch en quelques secondes
5. Les règles de détection Kibana évaluent les événements entrants — les correspondances génèrent des alertes dans Security
6. `Invoke-AtomicTest -Cleanup` supprime les artefacts de la VM

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

7/9 tests réussis. Un conteneur `remote` dédié (Ubuntu + SSH + rsync) a été ajouté comme serveur de staging. rsync push/pull, scp push/pull, sftp push/pull, et curl download+execute ont tous réussi.
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
