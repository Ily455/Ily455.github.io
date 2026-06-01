---
title: "Déploiement SIEM complet"
date: 2023-06-15
draft: false
description: "Déploiement complet d'une stack SOC : Elastic Stack, Wazuh, Suricata, MISP, TheHive et Cortex sur 6 VMs — de la collecte de logs jusqu'à la gestion d'incidents."
tags: ["siem", "elastic", "wazuh", "suricata", "misp", "thehive", "blue-team", "detection"]
---

> Projet académique — février à juin 2023 à l'ENSA Oujda. Objectif : concevoir et déployer une infrastructure SOC complète et fonctionnelle, capable de détecter des menaces réelles, corréler des événements hôtes et réseau, et gérer des incidents de bout en bout.

---

## Architecture

Six VMs Ubuntu 20.04.4 LTS (sauf Fleet sur CentOS 8), toutes sur VirtualBox 7.0.6 en mode bridged sur le même LAN. Chaque composant tourne sur sa propre VM pour éviter les interférences.

```
VM1 — Wazuh v4.4.1          192.168.1.120   4 Go RAM   HIDS, gestion des agents
VM2 — Elasticsearch + Kibana v7.17.10
                             192.168.1.106   8 Go RAM   Indexation, tableaux de bord
VM3 — Logstash v7.17.10     192.168.1.203   2 Go RAM   Pipeline d'ingestion
VM4 — Fleet server v4.33    192.168.1.45    4 Go RAM   Gestion Elastic Agents (CentOS 8)
VM5 — Suricata v6.0.11      192.168.1.205   4 Go RAM   IDS/IPS réseau
VM6 — MISP v2.4.170
       TheHive v4.1.24-1
       Cortex v3.1.7-1       192.168.1.204   4 Go RAM   Threat intel + gestion d'incidents
```

Toutes les VMs fonctionnent en CLI uniquement. Les interfaces web (Kibana, dashboard Wazuh, TheHive, MISP) sont accessibles depuis la machine physique.

![Architecture réseau de la stack SIEM](/images/projects/siem/net1.png)

### Flux de données

```
Endpoints (Elastic Agents)
        │
        ▼
Fleet Server ──────────────────► Elasticsearch
        │                              │
        │                              ▼
Logstash (pipeline)              Tableaux de bord Kibana
        │
        ▼
Suricata (événements réseau) ───► Elasticsearch

Agents Wazuh ──► Serveur Wazuh ──► Indexeur Wazuh ──► Dashboard Wazuh
                                         │
                                         ▼
                               TheHive (cas) ◄──► MISP (IoCs) ◄──► Cortex (analyse)
```

---

## Composants

### Elastic Stack

- **Elasticsearch** : indexe et stocke tous les événements en JSON, distribués en shards
- **Kibana** : tableaux de bord pour les métriques système (CPU, mémoire, disque), alertes Suricata avec géolocalisation source/destination, événements de sécurité Wazuh
- **Logstash** : pipeline d'ingestion avec filtrage, normalisation et enrichissement
- **Elastic Agent + Fleet** : déploiement centralisé des agents et gestion des politiques de collecte

![Dashboard Kibana — vue du trafic réseau](/images/projects/siem/kibana1.png)

### Wazuh

Trois agents déployés : deux hôtes Ubuntu et une instance DVWA. Le serveur Wazuh enrichit les événements avec le framework MITRE ATT&CK et les exigences de conformité (PCI DSS, NIST 800-53).

Catégories de détection configurées : événements d'authentification, intégrité des fichiers, escalade de privilèges, audit des appels système.

![Monitoring des agents Wazuh](/images/projects/siem/wazuh-1.png)

### Suricata

Déployé en mode IDS/IPS sur sa propre VM. Utilise le ruleset Emerging Threats. Testé avec le script **tmNIDS** pour injecter du trafic malveillant simulé :

- User-agents malware HTTP
- Autorités de certification non fiables
- Requêtes DNS vers des domaines .onion Tor
- Téléchargement d'exécutables/DLLs via HTTP
- Simulation de scan SSH sortant
- Réponses DNS de sinkhole
- Patterns C2 connus (Cryptowall, SuperFish)

Les alertes Suricata sont transmises à Elasticsearch via l'intégration Fleet, visibles dans Kibana avec carte de géolocalisation.

![Alertes IDS Suricata dans Kibana](/images/projects/siem/suricata1.png)

### MISP + TheHive + Cortex

- **MISP** : dépôt d'IoCs et partage de threat intelligence. Alimente l'enrichissement des analyses.
- **TheHive** : gestion des cas d'incidents. Les alertes Wazuh et Suricata escaladées créent des cas avec observables.
- **Cortex** : analyse automatisée des observables (IPs, hashes, domaines) via des analyseurs. Intégré à MISP pour la corrélation d'IoCs.

![Interface de gestion d'incidents TheHive](/images/projects/siem/3-thehive.png)

---

## Validation

Tests structurés en trois niveaux :

**Niveau 1 — Métriques hôtes** : les Elastic Agents remontent CPU, mémoire et disque vers Elasticsearch. Vérifié via le dashboard Kibana Metrics System.

**Niveau 2 — Détection réseau** : script tmNIDS exécuté sur la VM Suricata pour générer des patterns de trafic malveillant. Suricata a déclenché des alertes sur l'ensemble des catégories injectées :

| Signature | Catégorie |
|---|---|
| ET MALWARE Cryptowall .onion Proxy | Trojan réseau détecté |
| ET MALWARE SuperFish Possible | Trojan réseau détecté |
| ET DNS Reply Sinkhole | Trojan réseau détecté |
| ET POLICY malware User-Agent | Tentative de fuite d'information |
| ET INFO Anonymous File Sharing | Trafic potentiellement malveillant |
| ET DNS Query for .su TLD | Trafic potentiellement malveillant |

**Niveau 3 — Détection d'intrusion hôte** : Wazuh surveille 3 agents dont DVWA. Après génération d'activité :

- 712 événements de sécurité détectés
- 481 échecs d'authentification
- 16 succès d'authentification
- Événements taggés avec les TTPs MITRE ATT&CK : T1078 (Valid Accounts), T1548.003 (Sudo and Sudo Caching), T1100.001

---

## Limites

**Faux positifs** : certains patterns légitimes (variations TLS, requêtes DNS internes) ont déclenché des alertes. Le tuning des règles de détection demande une baseline du trafic normal — difficile à obtenir dans un environnement lab.

**Contraintes de ressources** : 6 VMs simultanées sollicitent fortement les machines physiques. Elasticsearch seul nécessite 8 Go de RAM pour fonctionner correctement. Cette architecture est faite pour du cloud ou du matériel dédié.

---

## Ce que j'en retiens

La chaîne de l'événement brut à l'alerte actionnable est plus longue qu'elle n'y paraît. Elasticsearch indexe vite et Kibana visualise bien, mais la qualité du signal dépend entièrement de ce qu'on lui envoie et comment on le normalise. Suricata détecte fiablement les patterns connus — mais est aveugle à ce que les règles ne couvrent pas. Wazuh apporte le contexte hôte que les outils purement réseau ne voient pas.

MISP + TheHive + Cortex est ce qui transforme un SOC en outil d'investigation réel : sans gestion de cas structurée et enrichissement des IoCs, les alertes restent des alertes et ne deviennent jamais des enquêtes.

---

## Ressources

- [Documentation Elastic Stack](https://www.elastic.co/guide/index.html)
- [Documentation Wazuh](https://documentation.wazuh.com/)
- [Documentation Suricata](https://suricata.readthedocs.io/)
- [MISP project](https://www.misp-project.org/)
- [TheHive project](https://thehive-project.org/)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [tmNIDS — testeur NIDS](https://github.com/3CORESec/testmynids.org)
