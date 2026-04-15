---
title: "Réseau d'entreprise virtualisé — VMware ESXi"
date: 2024-01-15
draft: false
description: "Projet de groupe simulant un réseau d'entreprise virtualisé complet sur VMware ESXi 6.5 — routeur Ubuntu, DNS sur Windows Server 2022, DHCP, serveur web Apache, postes de supervision et d'administration, tous déployés depuis des masters."
tags: ["reseau", "esxi", "vmware", "linux", "dns", "dhcp", "infrastructure", "virtualisation"]
---

> Projet de groupe — M2 FSI, module Sécurité des systèmes d'informations, AMU 2023–2024. Objectif : simuler un réseau d'entreprise virtualisé complet sur VMware ESXi, couvrant l'installation de l'hyperviseur, le déploiement de VMs depuis des masters, la configuration des services réseau et la supervision.

---

## Vue d'ensemble

Le projet consistait à concevoir et déployer un réseau d'entreprise virtualisé complet depuis zéro sur VMware ESXi 6.5. Chaque composant — routeur, DNS, DHCP, serveur web, poste de supervision, poste d'administration et postes utilisateurs — tourne comme une VM sur le même hôte ESXi, géré via l'interface web ESXi Host Client.

L'approche de déploiement repose sur des VMs masters : un master Windows Server et un master Ubuntu sont configurés une fois, puis clonés pour déployer de nouvelles instances sans réinstaller depuis zéro à chaque fois.

---

## Infrastructure

![Topologie réseau virtualisé VMware ESXi](/images/projects/esxi/topology.svg)

**Hyperviseur** : VMware ESXi 6.5, géré via l'ESXi Host Client.

| VM | OS | Rôle |
|---|---|---|
| Ubuntu Master | Ubuntu | Image de base pour les instances Ubuntu |
| Windows Server Master | Windows Server 2022 | Image de base pour les instances Windows |
| Routeur_US | Ubuntu Server 20 | Routeur réseau / passerelle |
| DNS-WS | Windows Server 2022 | Serveur DNS |
| DHCP_US | Ubuntu Server 20 | Serveur DHCP |
| Web_US | Ubuntu Server 20 | Serveur web (Apache) |
| Poste supervision | Ubuntu | Poste de supervision / monitoring |
| Poste admin | Ubuntu Desktop 22.0 | Poste d'administration |
| Poste utilisateur | Windows 10 | Poste utilisateur |

---

## Masters et déploiement de VMs

Avant de déployer les services, deux VMs masters ont été construites — une Windows Server 2022, une Ubuntu. Chacune est une image OS complètement installée et configurée, clonable pour déployer rapidement toute nouvelle instance. Chaque VM de service est créée depuis le master approprié, puis configurée pour son rôle spécifique.

Cette approche reflète les pratiques de production : une image de référence validée, clonée à la demande plutôt que réinstallée à chaque fois.

---

## Services réseau

### Routeur — Ubuntu Server 20

La VM routeur tourne sous Ubuntu Server avec le forwarding IP activé, configuré pour router le trafic entre les segments réseau. Elle fait office de passerelle par défaut pour les autres VMs.

### DNS — Windows Server 2022

DNS déployé sur Windows Server 2022. Résout les noms d'hôtes internes pour que les VMs puissent se référencer par nom plutôt que par IP — nécessaire pour le serveur web et les workflows d'administration.

### DHCP — Ubuntu Server 20

Serveur DHCP sur Ubuntu Server 20, distribuant automatiquement les configurations IP aux postes de travail.

### Serveur web — Ubuntu Server 20 (Apache)

Apache déployé sur Ubuntu Server 20. Utilisé pour vérifier la connectivité bout en bout sur le réseau et simuler un service web de production accessible depuis les postes utilisateurs.

---

## Supervision et administration

### Poste de supervision

Une VM dédiée au monitoring — suivi de l'état et des métriques des services déployés sur l'infrastructure.

### Poste d'administration — Ubuntu Desktop 22.0

Un poste Ubuntu Desktop 22.0 dédié à la gestion de l'infrastructure : configuration des services, accès à l'ESXi Host Client, et réalisation des tâches administratives de manière centralisée.

---

## Ce que j'ai appris

**Virtualisation :**
- L'approche par VM master est pertinente opérationnellement — une image de référence configurée une fois, clonée à la demande. Réinstaller depuis zéro à chaque fois ne passe pas à l'échelle
- L'ESXi Host Client donne une vue unifiée de toutes les VMs, leur utilisation des ressources et leur état — utile pour déboguer quand quelque chose échoue silencieusement
- Le matériel AMD Ryzen nécessite spécifiquement ESXi 6.5 (pas la dernière version) ; la compatibilité matérielle compte avant de choisir une version d'hyperviseur

**Réseau :**
- Configurer Ubuntu Server comme routeur (forwarding IP + règles de routage manuelles) rend les mécanismes du routage concrets d'une façon qu'une appliance dédiée ne permet pas
- Le DNS est fondamental — sans lui, chaque service doit être adressé par IP, ce qui casse dès que la moindre adresse change
- Tester chaque couche indépendamment (routeur → DNS → DHCP → web) avant intégration accélère considérablement le débogage de la stack complète

**Travail en groupe :**
- Le format mode opératoire — pas à pas, illustré, avec les résultats attendus par action — est la bonne approche documentaire pour un déploiement d'infrastructure reproductible
- Travailler en parallèle sur des composants différents nécessite une coordination explicite sur le plan d'adressage IP et les conventions de nommage avant que quiconque commence à configurer

---

## Ressources

- [Documentation VMware ESXi](https://docs.vmware.com/en/VMware-vSphere/)
- [Documentation Ubuntu Server](https://ubuntu.com/server/docs)
- [Documentation Windows Server 2022](https://learn.microsoft.com/fr-fr/windows-server/)
