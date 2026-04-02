---
title: "Analyse de firmware Zyxel — Extraction, crack ZIP et récupération de credentials"
date: 2025-01-07
draft: false
description: "Analyse complète d'un firmware Zyxel : extraction squashfs avec binwalk, attaque par texte clair connu sur le chiffrement ZIP via pkcrack, et récupération des credentials par hashcat depuis shadow.basic."
tags: ["firmware", "reverse-engineering", "binwalk", "hashcat", "cryptographie", "embarque"]
---

> Lab académique — M2 FSI, module sécurité des applications. Cible : binaire firmware Zyxel (`455ABUJOCO.bin`). Objectif : extraire le système de fichiers, identifier les données sensibles et récupérer les credentials via des attaques cryptographiques.

---

## Environnement

**OS** : VM Ubuntu
**Outils** : `binwalk`, `pkcrack`, `hashcat`, Python (génération de wordlist)
**Cible** : `455ABUJOCO.bin` — firmware routeur/pare-feu Zyxel

---

## Étape 1 — Extraction du firmware

En partant du binaire firmware brut, `binwalk` a été utilisé pour identifier les systèmes de fichiers embarqués et les formats de compression :

```bash
binwalk 455ABUJOCO.bin
```

`binwalk` a identifié une signature de système de fichiers compressé dans le binaire. Extraction :

```bash
binwalk -e 455ABUJOCO.bin
```

Le système de fichiers extrait est une image **squashfs** — un système de fichiers compressé en lecture seule couramment utilisé dans les dispositifs Linux embarqués. Le montage révèle une arborescence Linux :

```
/zyxel/
/ftp/
/etc/
/bin/
/usr/
...
```

---

## Étape 2 — Découverte du fichier de configuration

En naviguant dans le système de fichiers extrait, le fichier cible était `/etc/zyxel/sys/system-default.conf` — un fichier de configuration de 27 Ko contenant les paramètres du dispositif, les configurations des services et les credentials.

Ce fichier apparaît au même chemin dans le firmware qu'à l'intérieur de l'archive ZIP fournie avec le firmware — une observation clé pour l'étape suivante.

---

## Étape 3 — Crack du ZIP (attaque par texte clair connu)

Le firmware est distribué sous forme d'archive ZIP avec des contenus chiffrés. Le chiffrement utilise le chiffrement PKZIP legacy — un chiffrement de flux vulnérable à une attaque par texte clair connu si un contenu en clair est disponible.

Puisque `system-default.conf` a déjà été extrait du binaire brut et existe à un chemin connu dans le ZIP, ce fichier est le texte clair. `pkcrack` réalise l'attaque :

```bash
pkcrack -C firmware_chiffre.zip \
        -c "etc/zyxel/sys/system-default.conf" \
        -P archive_clair.zip \
        -p "etc/zyxel/sys/system-default.conf" \
        -d firmware_dechiffre.zip \
        -a
```

Les 5 paramètres :
- `-C` — ZIP chiffré cible
- `-c` — chemin du fichier connu dans le ZIP chiffré
- `-P` — ZIP contenant le texte clair connu
- `-p` — chemin du fichier connu dans le ZIP en clair
- `-d` — ZIP déchiffré en sortie
- `-a` — trouver toutes les clés valides

`pkcrack` a récupéré les clés de chiffrement du ZIP. L'archive ZIP a été entièrement déchiffrée.

**Pourquoi ça fonctionne** : le chiffrement PKZIP traditionnel utilise trois clés 32 bits initialisées depuis le mot de passe. Si un attaquant dispose d'environ 13 octets de texte clair connu à un offset connu, le keystream peut être récupéré via une attaque de type meet-in-the-middle. Ce schéma de chiffrement est cassé depuis 1994 (Biham & Kocher).

---

## Étape 4 — Récupération des credentials

À l'intérieur de l'archive déchiffrée, dans `/etc/zyxel/sys/`, deux fichiers ont été trouvés :

- `passwd.basic` — entrées de noms d'utilisateurs
- `shadow.basic` — mots de passe hashés (MD5Crypt, préfixe `$1$`)

Le format de hash est `$1$` (MD5Crypt, 500 itérations de MD5). Pour casser :

1. Génération d'une wordlist ciblée avec Python basée sur les patterns de mots de passe Zyxel connus
2. Exécution de `hashcat` en mode `-m 500` (MD5Crypt) :

```bash
hashcat -m 500 shadow.basic wordlist.txt
```

**Résultat** : mot de passe craqué — `Pr0w!aN_fXp`

Le crack a réussi en quelques secondes. MD5Crypt avec 500 itérations offre une résistance minimale face aux GPU modernes.

---

## Bilan des vulnérabilités

### 1. Chiffrement ZIP faible (classe CVE : Faiblesse cryptographique)

Le firmware utilise le chiffrement PKZIP legacy — un chiffrement de flux cassé en 1994. Une attaque par texte clair connu récupère l'intégralité de l'archive avec un seul fichier présent à la fois dans le ZIP chiffré et dans le binaire brut.

**Correction** : utiliser un ZIP chiffré AES-256 (WinZip AES ou 7-Zip AES), ou signer et chiffrer le firmware avec de la cryptographie asymétrique.

### 2. Credentials sensibles dans des fichiers de configuration accessibles

`shadow.basic` et `passwd.basic` sont stockés dans l'archive firmware sans protection supplémentaire au-delà du chiffrement ZIP (maintenant cassé). Une fois le ZIP déchiffré, les credentials sont directement accessibles.

**Correction** : ne pas stocker de credentials en clair ou faiblement hashés dans les images firmware. Utiliser des clés spécifiques au dispositif dérivées au moment du provisionnement.

### 3. Hashage de mots de passe faible

MD5Crypt (`$1$`) est déprécié. 500 itérations représentent un coût négligeable pour un attaquant disposant de GPU.

**Correction** : utiliser bcrypt, scrypt ou Argon2 pour le stockage des mots de passe.

---

## Résumé de la chaîne d'attaque

```
455ABUJOCO.bin
    │
    ├── binwalk -e → extraction squashfs
    │       └── /etc/zyxel/sys/system-default.conf (27 Ko)
    │
    ├── pkcrack (texte clair connu sur PKZIP)
    │       └── firmware_dechiffre.zip
    │               └── shadow.basic, passwd.basic
    │
    └── hashcat -m 500 → Pr0w!aN_fXp
```

---

## Ce que j'en retiens

- L'analyse de firmware commence avec `binwalk` — il identifie les signatures de compression et de système de fichiers indépendamment de l'extension du fichier
- Le chiffrement PKZIP legacy est fondamentalement cassé — un seul fichier connu dans l'archive suffit à récupérer toutes les clés
- MD5Crypt n'est pas un schéma de hashage sûr pour les modèles de menaces modernes — le coût est trop faible
- Les fichiers de configuration dans les firmwares contiennent souvent plus que prévu — credentials par défaut, tokens de service, configuration réseau
- La chaîne d'attaque (extraire → identifier le chevauchement → texte clair connu → crack de hash) est entièrement reproductible avec des outils publics

---

## Ressources

- [binwalk](https://github.com/ReFirmLabs/binwalk)
- [pkcrack](https://github.com/keyunluo/pkcrack)
- [hashcat](https://hashcat.net/hashcat/)
- [Biham & Kocher — A Known Plaintext Attack on the PKZIP Stream Cipher (1994)](https://link.springer.com/chapter/10.1007/3-540-60590-8_12)
- [Analyse de la faiblesse de MD5Crypt](https://www.usenix.org/legacy/events/usenix99/provos/provos.pdf)
