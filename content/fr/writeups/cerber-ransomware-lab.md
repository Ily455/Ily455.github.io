---
title: "Cerber Ransomware — Analyse Statique & Dynamique en Lab"
date: 2026-03-06
draft: false
description: "Analyse complète du ransomware Cerber sous Flare-VM et Cuckoo Sandbox — analyse statique du PE, traçage comportemental et indicateurs de compromission."
tags: ["analyse-malware", "ransomware", "reverse-engineering", "flare-vm", "cuckoo"]
---

> Travail réalisé lors de mon stage chez **Techso Group** (Casablanca, juillet–août 2023). Mission : étudier les méthodologies d'analyse de malwares et effectuer une analyse pratique du ransomware Cerber sous Flare-VM et Cuckoo Sandbox.

---

## Qu'est-ce que Cerber

Cerber est une famille de ransomware-as-a-service (RaaS), active principalement entre 2016 et 2017. C'est l'une des premières à avoir utilisé un moteur text-to-speech pour annoncer l'infection à la victime. Son modèle d'affiliation l'a rendu massivement distribué.

Malgré son ancienneté, c'est une cible d'apprentissage pertinente. Les techniques qu'il utilise — obfuscation d'API, chiffrement hybride, C2 via Tor — sont toujours présentes dans les familles modernes. L'objectif de ce lab n'était pas de trouver quelque chose de nouveau sur Cerber, mais de construire et pratiquer un workflow d'analyse reproductible.

---

## Mise en place du lab

### Analyse statique — Flare-VM

[Flare-VM](https://github.com/mandiant/flare-vm) est une distribution Windows dédiée à l'analyse de malwares, maintenue par Mandiant.

1. VM Windows 10 fraîche — **isolée, adaptateur hôte uniquement, pas d'accès internet**
2. Désactiver Windows Defender et les mises à jour automatiques avant l'installation
3. Exécuter le script d'installation Flare-VM — il tire les outils via Chocolatey
4. **Snapshot de l'état propre** avant de toucher tout échantillon

Outils principaux utilisés :

| Outil | Usage |
|-------|-------|
| Detect-It-Easy (DIE) | Identification packer/compilateur |
| PEStudio | En-tête PE, imports, chaînes, entropie |
| Ghidra | Désassemblage et décompilation |
| x64dbg | Débogage dynamique |
| FLOSS | Extraction de chaînes obfusquées |
| CFF Explorer | Visualisation structure PE |

> 📷 *[Placeholder — bureau Flare-VM]*

### Analyse dynamique — Cuckoo Sandbox

Cuckoo tourne sur un hôte Linux et lance des VMs Windows isolées pour exécuter les échantillons automatiquement.

```
Hôte Ubuntu (Cuckoo) → KVM → VM Windows 7 (agent Cuckoo)
```

Points clés de configuration :
- Serveur de résultats sur une interface joignable depuis la VM invitée
- `machinery = kvm` dans `cuckoo.conf`
- Réseau routé via InetSim — l'échantillon reçoit des réponses mais sans connectivité réelle
- Snapshot pris après installation de l'agent

```bash
cuckoo submit --timeout 120 cerber_sample.exe
```

---

## Analyse statique

### 1. Triage initial

Premier passage avec DIE avant tout :

```
Type de fichier : PE32 executable (GUI) Intel 80386
Packer          : UPX (certains samples) / packer custom
Compilateur     : MSVC
Entropie        : Élevée dans la section code — payload chiffré
```

L'entropie élevée dans la section `.text` est le premier indicateur — un logiciel légitime dépasse rarement 7.0.

### 2. Table des imports

Après dépaquetage, la table des imports devient lisible :

| Import | Capacité |
|--------|---------|
| `CryptGenKey`, `CryptEncrypt` | Chiffrement de fichiers via Windows CryptoAPI |
| `RegCreateKeyExW`, `RegSetValueExW` | Persistance registre |
| `CreateProcessW`, `ShellExecuteW` | Création de processus |
| `InternetOpenW`, `InternetConnectW` | Réseau / C2 |
| `FindFirstFileW`, `FindNextFileW` | Énumération du système de fichiers |
| `DeleteFileW` | Suppression de fichiers |

Crypto + registre + énumération FS = indicateurs forts de ransomware.

> 📷 *[Placeholder — table d'imports PEStudio]*

### 3. Extraction de chaînes avec FLOSS

`strings` standard rate les stack strings et les blobs encodés. FLOSS les résout :

```
.cerber                              ← extension ajoutée aux fichiers chiffrés
DECRYPT MY FILES                     ← base du nom des notes de rançon
hxxp://[redacté].onion/[victim_id]  ← C2 (service caché Tor)
vssadmin Delete Shadows /All /Quiet  ← suppression des clichés instantanés
bcdedit /set {default} recoveryenabled No
```

### 4. Schéma de chiffrement (Ghidra)

Cerber utilise un **chiffrement hybride** :

```
1. Au démarrage   → Génération clé AES-256 de session (CryptGenKey)
2. Par fichier    → Énumération FindFirstFile/FindNextFile
                  → Lecture du contenu
                  → Chiffrement AES-256
                  → Écriture du contenu chiffré
                  → Renommage : fichier.docx → fichier.docx.cerber
3. Clé en séquestre → Clé AES chiffrée avec la clé publique RSA-2048 de l'attaquant
                    → Envoi de la clé chiffrée au C2
4. Fichiers originaux → Supprimés
```

Sans la clé privée RSA du serveur C2, le déchiffrement est infaisable.

> 📷 *[Placeholder — vue décompilateur Ghidra, boucle de chiffrement]*

---

## Analyse dynamique — Rapport Cuckoo

### Système de fichiers

```
Créé :    %APPDATA%\{GUID_aléatoire}\cerber.exe   ← copie pour persistance
Créé :    Bureau\DECRYPT MY FILES.txt
Créé :    Bureau\DECRYPT MY FILES.html
Modifié : Tous les fichiers des répertoires utilisateur
Renommé : *.docx → *.docx.cerber
```

### Registre

```
HKCU\Software\Microsoft\Windows\CurrentVersion\Run
  "{nom_aléatoire}" = "%APPDATA%\{GUID}\cerber.exe"
```

Exécution à chaque connexion jusqu'au paiement de la rançon.

### Réseau

```
Requête DNS : [sous-domaine_aléatoire].top
Trafic UDP sur le port 6893
```

Une des signatures de Cerber : utilisation de **UDP port 6893** pour le C2, pas HTTP. Contourne les blocages HTTP simples.

### Processus créés

```
cmd.exe /c vssadmin Delete Shadows /All /Quiet
cmd.exe /c bcdedit /set {default} recoveryenabled No
cmd.exe /c bcdedit /set {default} bootstatuspolicy ignoreallfailures
```

> 📷 *[Placeholder — arbre des processus Cuckoo]*

---

## Indicateurs de Compromission

| Type | Valeur |
|------|--------|
| Extension fichier | `.cerber` |
| Notes de rançon | `DECRYPT MY FILES.txt`, `.html`, `.url` |
| Clé de registre | `HKCU\...\Run` avec nom GUID aléatoire |
| Protocole C2 | UDP port 6893 |
| Suppression VSS | `vssadmin Delete Shadows /All /Quiet` |
| Chemin persistance | `%APPDATA%\{GUID_aléatoire}\cerber.exe` |

---

## Ce que j'en retiens

**Technique :**
- Comment monter un environnement d'analyse isolé qui ne compromet pas l'hôte
- La différence entre triage statique (rapide, sans risque) et analyse dynamique (plus complète, plus risquée)
- Pourquoi le chiffrement hybride rend le déchiffrement infaisable sans la clé privée
- Les limites de Cuckoo : certains samples détectent le sandbox (uptime, résolution d'écran, mouvements souris) et restent dormants

**Méthodologie :**
- Toujours commencer par le statique — comprendre ce que le binaire *peut* faire avant de l'exécuter
- FLOSS extrait significativement plus de chaînes que `strings`
- Snapshot avant chaque exécution
- Les captures réseau (même simulées) donnent les patterns C2 avant de toucher un vrai sample

---

## Ressources

- [Mandiant FLARE-VM](https://github.com/mandiant/flare-vm)
- [FLOSS — extraction de chaînes obfusquées](https://github.com/mandiant/flare-floss)
- [Documentation Cuckoo Sandbox](https://cuckoo.sh/docs/)
- [MalwareBazaar](https://bazaar.abuse.ch/)
- [Any.run](https://app.any.run/)
- Rapports Kaspersky, Cisco Talos, Trend Micro sur Cerber (2016–2017)