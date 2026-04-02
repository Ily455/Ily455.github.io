---
title: "Architecture réseau sécurisée sur ESXi"
date: 2026-03-06
draft: false
description: "Conception et déploiement d'un réseau entreprise segmenté sur VMware ESXi — VLANs, pare-feu pfSense avec ACLs strictes, DMZ, VPN et périmètre de sécurité supervisé — entièrement virtualisé."
tags: ["reseau", "esxi", "vmware", "pfsense", "vlan", "pare-feu", "infrastructure"]
---

## Vue d'ensemble

Ce projet consistait à concevoir et déployer une architecture réseau complète de niveau entreprise, tournant entièrement sur VMware ESXi. L'accent était mis sur la sécurité : segmentation réseau par VLANs, pare-feu pfSense avec contrôle d'accès strict, une DMZ correcte pour les services exposés, et une passerelle VPN pour l'accès distant.

Tout ce qui nécessiterait normalement des commutateurs physiques, des routeurs et des appliances pare-feu a été virtualisé — ce qui a aussi permis de tester des scénarios d'attaque et des changements de règles pare-feu sans risquer d'infrastructure physique.

---

## Architecture

```
Internet (WAN)
      │
      ▼
┌─────────────┐
│   pfSense   │  ← Pare-feu, routeur, passerelle VPN, IDS/IPS
└──────┬──────┘
       │
   ┌───┴────────────────────────────────────┐
   │              vSwitch (ESXi)            │
   └───┬──────────┬──────────┬──────────┬──┘
       │          │          │          │
  VLAN 10    VLAN 20    VLAN 30    VLAN 99
  LAN/Corp   DMZ        Mgmt       Isolé
       │          │          │          │
  ┌────┴───┐ ┌────┴───┐ ┌────┴───┐ ┌────┴───┐
  │Postes  │ │Srv web │ │ESXi   │ │Sandbox │
  │AD DC   │ │Srv mail│ │vCenter│ │VMs     │
  │Srv fich│ │        │ │       │ │        │
  └────────┘ └────────┘ └────────┘ └────────┘
```

---

## Segments réseau

### VLAN 10 — LAN entreprise

Postes et serveurs internes. Ce segment a :
- Accès aux ressources internes (serveur de fichiers, AD)
- Accès internet sortant via pfSense (HTTP/HTTPS uniquement par défaut)
- Pas d'accès direct à la DMZ ou aux segments de gestion
- Contrôleur de domaine Active Directory pour l'authentification

### VLAN 20 — DMZ

Services devant être accessibles depuis internet. Règles strictes :
- Internet → DMZ : uniquement les ports 80 et 443 (web), 25/465/587/993 (mail)
- DMZ → LAN : **bloqué** — un hôte DMZ compromis ne peut pas atteindre les ressources internes
- DMZ → DMZ : contrôlé — les services peuvent communiquer sur des ports spécifiques uniquement

### VLAN 30 — Gestion

Hôte ESXi, vCenter, interface de gestion pfSense. Segment le plus restreint :
- Accessible uniquement depuis un poste admin dédié sur VLAN 10
- Pas de connexions entrantes depuis la DMZ ou internet
- Identifiants séparés du domaine entreprise

### VLAN 99 — Isolé / Sandbox

Pas de routage vers aucun autre segment. Utilisé pour :
- Tester de nouvelles VMs avant déploiement
- Exécuter des samples potentiellement malveillants (isolé de tout)
- Expérimentation en lab

---

## Configuration pfSense

### Règles pare-feu — politiques clés

Le jeu de règles pfSense par défaut a été remplacé par des règles allow explicites et une politique default-deny.

**Interface WAN :**
```
allow  TCP  any → DMZ:80,443         ← services web publics
allow  TCP  any → DMZ:25,465,587,993 ← mail
allow  UDP  any → WAN:1194           ← VPN
block  any  any → any                ← tout le reste
```

**LAN → WAN :**
```
allow  TCP  LAN → any:80,443    ← HTTP/HTTPS
allow  UDP  LAN → any:53        ← DNS
block  any  LAN → any           ← tout le reste (pas de ports arbitraires sortants)
```

**DMZ → LAN :**
```
block  any  DMZ → LAN           ← la DMZ ne peut pas atteindre le réseau interne
```

**Mgmt → any :**
```
allow  TCP  admin_ws → pfSense:443   ← interface web
allow  TCP  admin_ws → ESXi:443      ← gestion ESXi
block  any  → Mgmt:any              ← pas d'entrant vers le segment de gestion
```

### NAT

Le NAT sortant masquerade le trafic LAN et DMZ derrière l'IP WAN. La redirection de port sur WAN route le trafic entrant vers l'hôte DMZ approprié.

### VPN — OpenVPN

Accès distant via OpenVPN sur UDP 1194. Configuration :
- Authentification par certificat (pas d'auth par mot de passe seul)
- Les clients distants arrivent dans un sous-réseau VPN dédié
- Split tunneling désactivé — tout le trafic est routé via le VPN (le trafic qui n'a pas besoin d'être sécurisé ne devrait pas passer par le réseau entreprise)
- Les clients VPN accèdent uniquement aux ressources VLAN 10, avec les mêmes règles que le LAN local

### IDS/IPS — Snort sur pfSense

Snort tourne en mode inline sur l'interface WAN pour inspecter le trafic entrant :
- Ensembles de règles : ET Open, règles communautaires Snort
- Alertes sur les patterns d'exploit connus, les scans de ports, les paquets malformés
- Mode blocage pour les règles à haute confiance

> 📷 *[Placeholder — tableau de bord pfSense / vue des règles pare-feu]*

---

## Réseau virtuel ESXi

VMware ESXi gère le commutateur virtuel et le tagging VLAN. Chaque VM est connectée à un groupe de ports correspondant à un VLAN.

```
NIC physique ESXi (uplink)
    │
    └── vSwitch0
         ├── Port Group : "LAN"     (VLAN 10)
         ├── Port Group : "DMZ"     (VLAN 20)
         ├── Port Group : "Mgmt"    (VLAN 30)
         └── Port Group : "Sandbox" (VLAN 99)
```

pfSense tourne comme une VM avec des interfaces virtuelles connectées à chaque groupe de ports — il voit quatre interfaces réseau et fait office de routeur inter-VLAN et de pare-feu pour tout le trafic.

---

## Tests de sécurité

Avec l'environnement entièrement virtualisé, il était possible d'exécuter des scénarios d'attaque sans risque :

**Scan de ports depuis DMZ vers LAN :**
```bash
nmap -sS 192.168.10.0/24   ← depuis une VM en DMZ
# Tous les hôtes devraient apparaître comme "filtered" — pfSense bloque
```

**Tentative de mouvement latéral :**
Simulation d'un serveur web DMZ compromis essayant d'atteindre le contrôleur de domaine AD — toutes les connexions bloquées, alertes Snort déclenchées.

**Test de changement de règles pare-feu :**
Ajouter une règle, vérifier qu'elle autorise le trafic prévu, vérifier qu'elle n'ouvre rien d'involontaire, puis documenter et valider.

---

## Ce que j'ai appris

**Réseau :**
- La segmentation VLAN est le fondement — si votre réseau est plat, toute compromission devient une compromission totale
- La politique pare-feu default-deny est la seule approche sensée — default-allow avec des exceptions a toujours des lacunes
- Conception DMZ : l'invariant clé est qu'un hôte DMZ compromis ne peut pas atteindre directement les ressources internes

**Virtualisation :**
- Le réseau virtuel ESXi est un réseau entièrement défini par logiciel — vSwitch, groupes de ports et tagging VLAN fonctionnent comme du matériel physique
- Faire tourner le pare-feu comme une VM sur le même hôte qu'il protège est une limitation de conception — en production, le pare-feu devrait être sur du matériel séparé ou un hôte ESXi distinct

**Opérations de sécurité :**
- Des règles IDS sans tuning génèrent trop de bruit — la configuration Snort a nécessité le même type de tuning d'alertes que les règles SIEM
- Chaque règle pare-feu devrait avoir une justification documentée et une date de révision
- L'accès au segment de gestion est la cible à plus haute valeur — protéger vCenter et ESXi est aussi important que protéger les données

---

## Ressources

- [Documentation pfSense](https://docs.netgate.com/pfsense/en/latest/)
- [Documentation VMware ESXi](https://docs.vmware.com/en/VMware-vSphere/)
- [OpenVPN sur pfSense](https://docs.netgate.com/pfsense/en/latest/vpn/openvpn/)
- [Documentation des règles Snort](https://www.snort.org/documents)
- [Réseau virtuel VMware — concepts](https://www.vmware.com/topics/glossary/content/virtual-networking.html)
