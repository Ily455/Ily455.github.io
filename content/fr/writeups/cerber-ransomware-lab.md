---
title: "Cerber Ransomware — Analyse Statique & Dynamique en Lab"
date: 2023-09-01
draft: false
description: "Analyse complète d'un sample Cerber 2017 sous Flare-VM et Cuckoo Sandbox — version packée et dépaquetée, traçage comportemental du chargement DLL jusqu'au chiffrement, IoCs réels."
tags: ["analyse-malware", "ransomware", "reverse-engineering", "flare-vm", "cuckoo"]
---

> Travail réalisé lors de mon stage chez **Techso Group** (Casablanca, juillet–août 2023). Mission : étudier les méthodologies d'analyse de malwares et effectuer une analyse pratique du ransomware Cerber. Deux passes : automatisée avec Cuckoo d'abord pour orienter l'analyse, puis manuelle sous Flare-VM.

---

## Qu'est-ce que Cerber

Cerber est une famille de ransomware-as-a-service (RaaS), active principalement entre 2016 et 2017. Son modèle d'affiliation l'a rendu massivement distribué. Les techniques qu'il utilise — chargement dynamique d'API, chiffrement hybride, C2 sur canaux non-HTTP — sont toujours présentes dans les familles modernes. L'objectif n'était pas de trouver de nouvelles informations sur Cerber, mais de construire un workflow d'analyse reproductible sur un sample bien documenté.

---

## Mise en place du lab

**Flare-VM** (analyse dynamique manuelle) : VM Windows 10 fraîche, adaptateur hôte uniquement, pas d'internet. Windows Defender et mises à jour automatiques désactivés avant l'installation.

**Cuckoo Sandbox** (pré-analyse automatisée) : hôte Ubuntu → KVM → VM Windows 7 avec agent Cuckoo. Réseau routé via InetSim. Exécuté en premier pour identifier les points chauds comportementaux.

Outils principaux :

| Outil | Usage |
|-------|-------|
| Detect-It-Easy (DIE) / exeinfope | Détection packer/compilateur |
| PEStudio | En-tête PE, imports, entropie, chaînes |
| Ghidra | Désassemblage et décompilation |
| x64dbg | Débogage dynamique |
| FLOSS | Extraction de chaînes obfusquées |
| unpac.me | Dépaquetage automatisé |

---

## Partie I — Sample packé

### Informations fichier

```
Nom        : cerber.exe
Type       : PE32 (GUI) Intel 80386
Taille     : 619 008 octets (622 592 sur disque)
Timestamp  : Wed May 24 20:48:34 2017 UTC
MD5        : 8b6bc16fd137c09a08b02bbe1bb7d670
SHA-1      : c69a0f6c6f809c01db92ca658fcf1b643391a2b7
SHA-256    : e67834d1e8b38ec5864cfa101b140aeaba8f1900a6e269e6a94c90fcbfe56678
Compilateur: Microsoft Visual C/C++ (2008)
Linker     : Microsoft Linker 9.0 [GUI32]
```

### Structure PE

Sections : `.text` 52,19% (exécutable), `.data` 0,74% (inscriptible), `.rdata` 38,79%, `.rsrc` 8,11%

La section `.rsrc` contient une icône mais pas de ressources significatives. Les informations de relocation sont supprimées des en-têtes.

### Détection du packing

DIE et exeinfope ne signalent aucun packer connu — ce qui est trompeur. Le désassemblage de `.text` révèle très peu de code réel. Le comportement véritable n'apparaît qu'à l'exécution via le pattern d'appels API : `LdrGetProcedureAddress` est utilisé pour résoudre dynamiquement `LoadLibraryExA`, `GetProcAddress`, `VirtualAlloc` et `VirtualProtect`. C'est la signature d'un stub de dépaquetage custom. Le sample a été dépaqueté via **unpac.me**.

### Table des imports (version packée)

Même avant dépaquetage, la table des imports est large et suspecte :

- Opérations fichiers : `CreateFile`, `CopyFile`, `DeleteFile`, `MoveFile`, `FindFirstFile`, `FindNextFile`, `GetFileSize`
- Registre : `RegCreateKeyEx`, `RegDeleteKey`, `RegOpenKeyEx`, `RegSetValueEx`, `RegQueryValueEx`
- Processus : `CreateProcess`, `CreateProcessAsUser`, `ShellExecuteEx`, `TerminateProcess`
- Mémoire : `VirtualAlloc`, `VirtualFree`, `VirtualQuery`
- Crypto : accédé via chargement dynamique (absent de la table statique)

---

## Partie II — Payload dépaqueté

### Informations fichier

```
Nom        : 53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924
Taille     : 196 608 octets
Timestamp  : Thu May 18 09:35:03 2017
MD5        : ab0953388bf5b63bbc85cd10a7b73022
SHA-1      : 5fa927f92c01364182a93d7468ba4416802cb722
SHA-256    : 53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924
Compilateur: Microsoft Visual C/C++ (2010 SP1)
Linker     : Microsoft Linker 10.0 [GUI32]
```

Compilateur différent de la version packée — le stub de packing et le payload ont été compilés séparément.

### Structure PE (dépaqueté)

Sections : `.text` 28,13%, `.rdata` 2,34%, `.data` 66,67% (inscriptible), `.reloc` 2,34%

La section `.rdata` a une entropie de **7,271** — anormalement élevée, indiquant des données obfusquées ou chiffrées en lecture seule. La section `.data` à 66,67% est anormalement large pour des données inscriptibles.

### Imports statiques (dépaqueté)

La table des imports statiques ne contient que `ntdll.dll` : `memmove`, `memset`, `memcpy`, `NtQueryVirtualMemory`. Tout le reste est chargé à l'exécution.

Les chaînes dans le binaire référencent des bibliothèques absentes de la table d'imports : `crypt32.dll`, `kernel32.dll`, `ntdll.dll`, `urlmon.dll`, `advapi32.dll` — toutes chargées dynamiquement via `LdrLoadDll`.

---

## Analyse dynamique

### Séquence d'exécution (Cuckoo + Flare-VM)

**1. Chargement dynamique des bibliothèques**

Le malware appelle `LdrLoadDll` pour charger son jeu complet de dépendances à l'exécution : `advapi32`, `crypt32`, `IMM32`, `gdi32`, `mpr`, `netapi32`, `SAMCLI`, `NETUTILS`, `ole32`, `oleaut32`, `powrprof`, `shlwapi`. Aucun n'apparaît dans la table des imports statiques.

**2. Manipulation du pare-feu**

Avant toute action visible, Cerber lance `netsh.exe` pour affaiblir les défenses de l'hôte :

```
netsh.exe advfirewall set allprofiles state on
netsh.exe advfirewall reset
netsh.exe advfirewall firewall add rule name="FktwwYI8qV" dir=out action=block program="C:/Program Files/Windows Defender/MpCmdRun.exe"
netsh.exe advfirewall firewall add rule name="OpnMEgcJzO" dir=out action=block program="C:/Program Files/Windows Defender/MSASCui.exe"
```

Les deux dernières règles bloquent le trafic sortant de l'outil de ligne de commande et de l'interface utilisateur de Windows Defender — empêchant les mises à jour de protection en temps réel et les rapports de menaces.

**3. Communication C2**

Cerber envoie des données chiffrées via **UDP port 6893** vers une plage d'adresses IP :
- 178.33.160.0 – 178.33.163.255
- 178.33.158.0 – 178.33.158.31
- 178.33.159.0 – 178.33.159.31

Implémenté via `sendto` dans `ws2_32`. L'utilisation d'UDP (pas HTTP/HTTPS) contourne la supervision au niveau applicatif. `InternetCrackUrl` est aussi appelé pour parser les URLs d'API Bitcoin pour le suivi des paiements.

**4. Énumération du système de fichiers et chiffrement**

Le malware utilise `FindFirstFileExW`, `GetFileAttributesW`, `NtQueryDirectoryFile` pour énumérer le système de fichiers récursivement. Les répertoires par défaut comme `Pictures` sont ignorés — ciblage sélectif.

Pour chaque fichier :
```
NtReadFile        → lecture du contenu dans un buffer
CryptEncrypt      → chiffrement du contenu (key_handle: 0x004ca898)
NtWriteFile       → écriture des données chiffrées dans le fichier
MoveFileWithProgressW → renommage vers un nom et une extension aléatoires
```

![Log Cuckoo — appel API CryptEncrypt](/images/writeups/cerber/fig4.png)

Exemple observé : `c:\Acrobat3\Reader\ACROBAT.PDF` → `c:\Acrobat3\Reader\9Fck7YJuLS.af22`

![Log Cuckoo — NtWriteFile (sortie chiffrée)](/images/writeups/cerber/fig5.png)

Le fichier chiffré reçoit un **nom aléatoire** avec une **extension aléatoire** — pas `.cerber`. C'est une variante tardive avec comportement d'extension aléatoire pour contourner la détection par signature d'extension.

**5. Livraison de la note de rançon**

Après chiffrement de chaque répertoire, Cerber dépose deux artefacts :
- `_R_E_A_D___T_H_I_S___LHYR8QK_.hta` — HTML Application, lancé via `mshta.exe`
- `_R_E_A_D___T_H_I_S___DRIDMPY_.txt` — texte brut, ouvert via `notepad.exe`

Le fichier HTA affiche une page de rançon avec liens de paiement vers des services cachés Tor.

![Note de rançon Cerber sur le bureau infecté](/images/writeups/cerber/cuckoo-report.png)

**6. Suivi du paiement Bitcoin**

![Page de paiement Cerber via Tor](/images/writeups/cerber/cuckoo-network.png)

Avant de se terminer, Cerber interroge les APIs de blockchain Bitcoin via `InternetCrackUrl` :

```
http://api.blockcypher.com/v1/btc/main/addrs/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
http://btc.blockr.io/api/v1/address/txs/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
https://chain.so/api/v2/address/btc/17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt?_=...
```

Vérifie si la victime a payé l'adresse BTC `17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt` — sans contact C2 direct.

**7. Auto-terminaison**

```
taskkill /f /im "cerber.exe"
```

Cerber se termine lui-même après le chiffrement — stratégie délibérée pour limiter le temps d'inspection post-exécution.

---

## Techniques d'évasion

- **Packing custom** : le stub est indétecté par les scanners standards, reconstruit la table des imports à l'exécution via `LdrGetProcedureAddress`
- **Extensions aléatoires** : les variantes tardives abandonnent `.cerber` pour des noms et extensions aléatoires — casse la détection par extension
- **Injection de règles pare-feu** : bloque le trafic sortant de Windows Defender avant de commencer le chiffrement
- **Auto-terminaison** : tue son propre processus après chiffrement

---

## Indicateurs de Compromission

| Type | Valeur |
|------|--------|
| SHA-256 packé | `e67834d1e8b38ec5864cfa101b140aeaba8f1900a6e269e6a94c90fcbfe56678` |
| SHA-256 dépaqueté | `53779238d1cc9ceddc3cf8a0debbfc2f3888a9c3cbfb0cbae4b1ae40b4aa5924` |
| Extension fichier | Aléatoire (ex. `.af22`) + nom aléatoire |
| Note de rançon | `_R_E_A_D___T_H_I_S___[aléatoire].hta` / `.txt` |
| Protocole C2 | UDP port 6893 |
| Plage IP C2 | 178.33.158.0 – 178.33.163.255 |
| Domaines | `chain.so`, `bitaps.com`, `btc.blockr.io`, `api.blockcypher.com` |
| Adresse Bitcoin | `17gd1msp5FnMcEMF1MitTNSsYs7w7AQyCt` |
| Règles pare-feu | Bloc sortant sur `MpCmdRun.exe`, `MSASCui.exe` |

---

## Ce que j'en retiens

L'apport le plus utile n'est pas la liste d'IoCs — c'est le workflow. Cuckoo donne un point de départ en 10 minutes ; l'analyse manuelle sous Flare-VM remplit les détails.

La détection du packing a été une bonne leçon : exeinfope qui dit "non packé" ne signifie pas non packé. Le désassemblage et le pattern d'appels API à l'exécution sont des signaux plus fiables que les scanners statiques.

Le suivi du paiement via les APIs blockchain était inattendu — les attaquants peuvent vérifier le statut du paiement sans contact C2 direct, réduisant leur exposition opérationnelle.

---

## Ressources

- [Mandiant FLARE-VM](https://github.com/mandiant/flare-vm)
- [FLOSS — extraction de chaînes obfusquées](https://github.com/mandiant/flare-floss)
- [Documentation Cuckoo Sandbox](https://cuckoo.sh/docs/)
- [unpac.me — dépaqueteur automatisé](https://www.unpac.me/)
- [MalwareBazaar](https://bazaar.abuse.ch/)
