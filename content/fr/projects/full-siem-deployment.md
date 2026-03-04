---
title: "Déploiement SIEM complet"
date: 2023-06-15
draft: false
tags: ["siem", "elastic", "wazuh", "misp", "thehive", "suricata"]
description: "Déploiement d’une stack complète orientée SOC pour la détection et l’analyse d’incidents."
---

Déploiement d’un environnement SIEM/SOC complet pour pratiquer l’ingénierie de détection, la corrélation d’événements et la gestion d’incidents.

## Stack

- Elastic Stack pour la collecte, l’indexation et la visualisation
- Wazuh pour la supervision hôte et la télémétrie sécurité
- Suricata pour la détection réseau
- MISP pour la gestion et l’enrichissement d’IoC
- TheHive pour l’investigation et le suivi de cas

## Ce sur quoi j’ai travaillé

- Construction et intégration de la chaîne complète, de la télémétrie à l’investigation
- Écriture et ajustement de règles de détection
- Corrélation d’événements entre données hôtes et réseau
- Mise en place de workflows d’analyse orientés IoC

## Ce que j’ai appris

- La valeur d’un SIEM dépend fortement de la qualité des données et de la logique de corrélation
- Le tuning de détection est itératif et dépend du contexte
- Les outils d’incident sont utiles uniquement si les workflows sont documentés et reproductibles
