---
title: "Modélisation des menaces ICS/OT — Trois attaques sur infrastructures critiques"
date: 2025-03-26
draft: false
description: "Analyse structurée de trois attaques ICS/OT via MITRE ATT&CK for ICS : Stuxnet (2010), réseau électrique ukrainien (2015), secteur énergétique danois (2023). Reconstruction des chaînes d'attaque, mapping des TTPs et comparaison inter-incidents."
tags: ["ics", "ot", "scada", "mitre-attack", "modelisation-menaces", "stuxnet", "infrastructure-critique"]
---

> Projet académique en groupe — AMU M2 FSI, 2023–2024. Thème : sécurité de la convergence IT/OT dans le secteur énergétique. Ce write-up couvre la méthodologie de modélisation des menaces appliquée à trois incidents ICS majeurs via MITRE ATT&CK for ICS.

---

## Contexte : convergence IT/OT

La Technologie Opérationnelle (OT) — systèmes SCADA, automates programmables (PLC), systèmes de contrôle industriels — était historiquement isolée en air gap. La convergence IT/OT (capteurs IoT, connectivité cloud, accès distant) a changé la donne : les mêmes chemins réseau qui améliorent l'efficacité opérationnelle créent une surface d'attaque.

L'asymétrie clé : les systèmes OT privilégient la disponibilité sur la confidentialité. Les contraintes de disponibilité limitent les opérations de patch. Les systèmes legacy fonctionnent pendant 20+ ans. Une vulnérabilité qui serait un incident mineur en IT peut causer des dommages physiques et des coupures de service en OT.

---

![Chaînes d'attaque ICS — Stuxnet, Ukraine 2015, SektorCERT 2023](/images/writeups/ics/attack-chain.svg)

## Méthodologie

Chaque incident a été mappé sur **MITRE ATT&CK for ICS** (distinct du framework entreprise). Tactiques clés pertinentes pour les attaques ICS :

| ID Tactique | Nom | Description |
|-------------|-----|-------------|
| TA0108 | Initial Access | Entrée dans le réseau IT ou OT |
| TA0104 | Execution | Exécution de code contrôlé par l'attaquant |
| TA0110 | Persistence | Maintien du foothold à travers les redémarrages |
| TA0111 | Privilege Escalation | Obtention d'accès élevés |
| TA0103 | Lateral Movement | Déplacement depuis l'IT vers l'OT |
| TA0102 | Collection | Collecte de données de process, configurations |
| TA0109 | Inhibit Response Function | Désactivation des systèmes de sécurité, alarmes |
| TA0105 | Impact | Perturbation physique, destruction |

---

## Cas 1 — Stuxnet (2010)

### Contexte

Stuxnet est la première cyberarme connue conçue pour causer des destructions physiques. Découvert en juin 2010, il ciblait les centrifugeuses nucléaires iraniennes à l'installation d'enrichissement de Natanz. Le ver exploitait **4 vulnérabilités zero-day** simultanément et contenait environ 150 000 lignes de code — d'un ordre de grandeur plus complexe que tout malware de l'époque.

### Chaîne d'attaque

**Accès initial — Support amovible (T0819)**
Stuxnet se propagait via des clés USB infectées. La cible (Natanz) était isolée d'internet — les médias physiques constituaient le seul vecteur d'entrée. Le ver exploitait une vulnérabilité Windows Shell LNK (CVE-2010-2568) déclenchée simplement en visualisant le contenu du lecteur — sans exécution utilisateur requise.

**Exécution — Quatre zero-days**
- CVE-2010-2568 : Windows Shell LNK (zero-day)
- CVE-2010-2772 : Windows Task Scheduler (zero-day)
- CVE-2010-2729 : Windows Print Spooler (zero-day)
- CVE-2010-2772 : Windows Server Service (zero-day)

**Persistance + Rootkit**
Stuxnet utilisait un rootkit (signé avec des certificats volés Realtek et JMicron) pour cacher ses fichiers et entrées de registre. Il modifiait la logique ladder des PLC tout en masquant sa présence à l'interface SCADA Siemens WinCC.

**Mouvement latéral — ciblage des PLC SIMATIC S7**
Le payload ciblait spécifiquement les PLC Siemens S7-315 et S7-417 connectés via le logiciel Siemens Step 7. Il interceptait les communications entre le poste ingénierie et les PLC.

**Impact — Destruction physique (T0879)**
La logique de contrôle des centrifugeuses a été modifiée pour faire tourner les rotors à des fréquences anormales (1 410 Hz puis 2 Hz, puis 1 064 Hz) tout en affichant une opération normale aux opérateurs. Environ 1 000 centrifugeuses ont été endommagées ou détruites. Aucune alarme n'était visible.

| Tactique ATT&CK | Technique | Détail |
|-----------------|-----------|--------|
| TA0108 Initial Access | T0819 Removable Media | Exploit LNK USB, contournement air gap |
| TA0104 Execution | T0807 Command-Line Interface | Quatre zero-days enchaînés |
| TA0110 Persistence | T0873 Project File Infection | Modification logique ladder + rootkit |
| TA0103 Lateral Movement | T0843 Program Download | Interception Step 7 |
| TA0109 Inhibit Response | T0838 Modify Alarm Settings | Masquage des lectures anormales |
| TA0105 Impact | T0879 Damage to Property | ~1 000 centrifugeuses détruites |

### Points clés

L'air gap n'était pas une frontière de sécurité — il a été contourné via la chaîne d'approvisionnement (clés USB infectées circulant parmi les sous-traitants). Le rootkit démontre qu'un adversaire suffisamment sophistiqué peut maintenir une dénégation plausible tout en causant des dommages physiques. Quatre zero-days simultanés signalent un acteur étatique avec des ressources importantes.

---

## Cas 2 — Réseau électrique ukrainien (décembre 2015)

### Contexte

Le 23 décembre 2015, des cyberattaques coordonnées ont coupé l'électricité à environ **225 000 clients** dans trois sociétés régionales de distribution ukrainiennes (Kyivoblenergo, Prykarpattyaoblenergo, Chernivtsioblenergo). Attribué à Sandworm (APT44, unité GRU 74455). Première cyberattaque confirmée ayant causé une panne électrique.

### Chaîne d'attaque

**Accès initial — Hameçonnage ciblé (T0865)**
Les employés ont reçu des emails ciblés contenant des documents Word malveillants avec le malware BlackEnergy. BlackEnergy 3 était livré via des macros.

**Exécution + Persistance**
BlackEnergy fournissait une backdoor persistante. Les attaquants ont maintenu l'accès pendant **plusieurs mois** avant la perturbation — conduisant une reconnaissance, récoltant des identifiants VPN, et cartographiant l'environnement SCADA.

**Mouvement latéral — pivot VPN**
Avec les identifiants VPN récoltés, les attaquants se sont déplacés du réseau IT vers le réseau OT. Le VPN ne disposait pas d'authentification multifacteur.

**Collecte**
Les attaquants ont récupéré des captures d'écran des opérateurs SCADA, des configurations de sous-stations, et appris les procédures opérationnelles normales. Ils ont attendu.

**Impact — Perturbation coordonnée (T0879)**
Le 23 décembre :
1. Accès simultané aux systèmes SCADA/HMI de 3 sociétés de distribution
2. Interfaces de contrôle des opérateurs saisies via UltraVNC remote desktop
3. 30 sous-stations mises hors ligne manuellement via l'interface SCADA
4. **KillDisk** déployé pour écraser le MBR des postes SCADA et RTU — rendant les systèmes non démarrables
5. Lignes téléphoniques saturées d'appels pour empêcher les opérateurs de signaler la panne

La combinaison manipulation SCADA + KillDisk + saturation téléphonique démontre une superposition délibérée d'impact et de suppression de réponse.

| Tactique ATT&CK | Technique | Détail |
|-----------------|-----------|--------|
| TA0108 Initial Access | T0865 Spearphishing Attachment | BlackEnergy 3 via macro Word |
| TA0110 Persistence | T0891 Hardcoded Credentials | Identifiants VPN récoltés |
| TA0103 Lateral Movement | T0859 Valid Accounts | Pivot VPN vers réseau OT |
| TA0102 Collection | T0801 Monitor Process State | Captures d'écran opérateurs |
| TA0109 Inhibit Response | T0804 Block Reporting Message | Saturation lignes téléphoniques |
| TA0105 Impact | T0879 Damage to Property | KillDisk, 30 sous-stations hors ligne |

### Points clés

L'absence de MFA sur le VPN était le pivot critique. Plusieurs mois de dwell time ont permis aux attaquants de comprendre l'environnement avant d'agir. KillDisk a été déployé non pour causer la panne mais pour ralentir la restauration — la panne elle-même a été réalisée via des opérations SCADA normales. L'attaque nécessitait à la fois une capacité technique et une planification opérationnelle.

---

## Cas 3 — Secteur énergétique danois (mai 2023)

### Contexte

En mai 2023, **22 entreprises énergétiques danoises** ont été attaquées dans une campagne coordonnée en deux vagues. Documenté par SektorCERT (CERT danois des infrastructures critiques). L'attaque exploitait trois vulnérabilités de pare-feux Zyxel patchées entre avril et mai 2023 — les attaquants ont agi avant que de nombreux opérateurs n'aient appliqué les correctifs.

### Vulnérabilités exploitées

| CVE | CVSS | Type | Produits affectés |
|-----|------|------|-------------------|
| CVE-2023-28771 | 9,8 | Injection de commande OS (non authentifié) | Zyxel ATP, USG FLEX, VPN, ZyWALL |
| CVE-2023-33009 | 9,8 | Buffer Overflow (pré-auth) | Mêmes gammes |
| CVE-2023-33010 | 9,8 | Buffer Overflow (pré-auth) | Mêmes gammes |

Les trois sont des vulnérabilités non authentifiées pré-auth — aucun identifiant requis pour les exploiter.

### Attaque en deux vagues

**Vague 1** : Exploitation opportuniste de CVE-2023-28771 sur 11 entreprises. Injection de commande via des paquets IKEv2 forgés. Les attaquants ont obtenu un accès shell aux pare-feux périmètre. Les systèmes de contrôle industriels de certaines entreprises étaient directement accessibles depuis les pare-feux compromis.

**Vague 2** (quelques jours après) : Suivi ciblé contre les entreprises avec accès réseau OT confirmé. La seconde vague utilisait l'infrastructure botnet Mirai et tentait du DDoS en parallèle de mécanismes de persistance sur les pare-feux. Plusieurs entreprises ont dû se déconnecter d'internet.

La réponse rapide de SektorCERT (coordination simultanée des 22 entreprises, partage d'IoC en temps réel) a limité le rayon d'explosion. Aucun impact physique confirmé sur les opérations du réseau, mais plusieurs entreprises ont perdu leur capacité de surveillance distante pendant l'incident.

| Tactique ATT&CK | Technique | Détail |
|-----------------|-----------|--------|
| TA0108 Initial Access | T0866 Exploitation of Remote Services | CVE-2023-28771 RCE non authentifié |
| TA0104 Execution | T0807 Command-Line Interface | Injection de commande via IKEv2 |
| TA0110 Persistence | T0891 Hardcoded Credentials | Mécanismes de persistance pare-feu |
| TA0103 Lateral Movement | T0843 Program Download | Pivot vers réseaux OT |
| TA0109 Inhibit Response | T0804 Block Reporting Message | DDoS + déconnexion internet forcée |
| TA0105 Impact | T0879 Damage to Property | Perte surveillance distante, accès OT potentiel |

### Points clés

Le délai de patch — 22 entreprises avec un firmware Zyxel vulnérable après la disponibilité des correctifs — illustre le défi opérationnel de la gestion des patches en infrastructure critique. La structure en deux vagues (scan opportuniste → suivi ciblé) est cohérente avec un modèle reconnaissance-puis-exploitation. La réponse coordonnée de SektorCERT est un cas de référence pour la réponse aux incidents au niveau sectoriel.

---

## Analyse transversale

| Dimension | Stuxnet | Ukraine 2015 | Danemark 2023 |
|-----------|---------|--------------|---------------|
| Acteur | État-nation | État-nation (Sandworm/GRU) | Inconnu (Sandworm suspecté) |
| Vecteur initial | Physique (USB) | Hameçonnage ciblé | CVE non patchés |
| Dwell time | Plusieurs mois | Plusieurs mois | Quelques jours |
| Méthode d'accès OT | Chaîne d'approvisionnement | Pivot VPN | Exploit pare-feu |
| Impact physique | Oui (~1 000 centrifugeuses) | Oui (225k clients) | Non (quasi-incident) |
| Réponse défenseurs | Retardée (opération couverte) | Restauration manuelle | CERT coordonné |

**Patterns communs aux trois cas :**
- La compromission du réseau IT précède l'impact OT
- Le dwell time est utilisé pour la reconnaissance avant l'action
- Les systèmes de sécurité/surveillance sont explicitement ciblés ou contournés
- La restauration est compliquée par des mécanismes anti-forensique ou de persistance délibérés

---

## Ce que révèle cette analyse

Mapper ces attaques sur MITRE ATT&CK for ICS révèle que malgré les années qui les séparent, les patterns d'attaque sont structurellement similaires : la frontière IT/OT est franchie par une méthode différente à chaque fois (USB, VPN, pare-feu), mais la séquence reconnaissance → impact est constante.

Le framework met également en évidence où les défenses auraient le plus d'effet : l'Initial Access (MFA, gestion des patches, contrôle des médias amovibles) et le Lateral Movement (segmentation réseau, renforcement de la frontière OT/IT) apparaissent dans les trois cas comme les points d'étranglement critiques.

---

## Ressources

- [MITRE ATT&CK for ICS](https://attack.mitre.org/matrices/ics/)
- [SektorCERT — Rapport sur le secteur énergétique danois (2023)](https://sektorcert.dk/wp-content/uploads/2023/11/SektorCERT-The-attack-against-Danish-critical-infrastructure-TLP-WHITE.pdf)
- [ICS-CERT — Analyse de l'attaque sur le réseau électrique ukrainien](https://www.cisa.gov/uscert/ics/alerts/IR-ALERT-H-16-056-01)
- [Langner — To Kill a Centrifuge (analyse technique Stuxnet)](https://www.langner.com/to-kill-a-centrifuge/)
- [IEC 62443 — Sécurité des systèmes d'automatisation et de contrôle industriels](https://www.iec.ch/iecnow/iecnow-smart-manufacturing/iec-62443/)
