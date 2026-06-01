---
title: "NIST CSF v2 ↔ ISO 27002:2022 — Notes de Mapping"
date: 2026-03-06
draft: false
description: "Notes d'un projet de correspondance entre le NIST Cybersecurity Framework v2 et les contrôles ISO 27002:2022, réalisé lors d'un stage GRC chez Enedis. Méthodologie, points d'alignement et outil de simulation de maturité construit sur le mapping."
tags: ["grc", "nist", "iso27001", "conformite", "gouvernance"]
---

> Travail réalisé lors de mon stage de fin d'études chez **Enedis** (Aix-en-Provence, avril–septembre 2025). Mission : produire une correspondance entre le NIST CSF v2 et ISO 27002:2022, puis construire un simulateur de maturité pour aider les équipes à évaluer leur posture de cybersécurité avec l'un ou l'autre référentiel.

---

## Pourquoi ce mapping est utile

Les organisations opérant à l'international font souvent face aux deux référentiels simultanément :

- **ISO 27001/27002** — dominant en Europe, requis pour la certification, structuré autour d'un SMSI
- **NIST CSF** — largement utilisé aux États-Unis, adoption croissante mondiale, plus orienté opérationnel

Une entreprise certifiée ISO 27001 peut aussi avoir besoin de reporter selon le NIST CSF pour des partenaires américains — ou l'inverse. Faire ça manuellement — lire les deux référentiels et tenter de trouver les correspondances — est fastidieux, source d'erreurs et incohérent selon les équipes.

Un mapping permet de répondre à : *"Si on score X sur ce contrôle ISO, qu'est-ce que ça implique pour notre sous-catégorie NIST ?"*

---

## Les référentiels

### NIST CSF v2

Le Cybersecurity Framework a été mis à jour en v2 en 2024. Il s'organise autour de **6 Fonctions** :

```
GOVERN (GV)   ← nouveau en v2 — contexte organisationnel, stratégie de gestion des risques
IDENTIFY (ID) ← gestion des actifs, évaluation des risques
PROTECT (PR)  ← contrôle d'accès, sécurité des données, formation
DETECT (DE)   ← anomalies, surveillance, processus de détection
RESPOND (RS)  ← réponse aux incidents, communications, analyse
RECOVER (RC)  ← planification de la reprise, améliorations
```

Chaque fonction contient des **Catégories** et **Sous-catégories** — 106 sous-catégories au total en v2.

### ISO 27002:2022

ISO 27002 est le code de bonnes pratiques qui soutient ISO 27001. La version 2022 a réorganisé les contrôles en **4 thèmes** :

```
Contrôles organisationnels  (37 contrôles)
Contrôles humains           (8 contrôles)
Contrôles physiques         (14 contrôles)
Contrôles technologiques    (34 contrôles)
```

93 contrôles au total. Chaque contrôle dispose d'attributs (type de contrôle, concepts de cybersécurité, capacités opérationnelles) qui facilitent la catégorisation et le mapping.

---

![Structure NIST CSF v2 et ISO 27002:2022 — mapping](/images/writeups/nist/framework-mapping.svg)

## Méthodologie

### Étape 1 — Ancrage sur les concepts cybersécurité partagés

Les deux référentiels s'appuient sur des concepts sous-jacents similaires. ISO 27002:2022 tague explicitement chaque contrôle avec des concepts cybersécurité issus du NIST CSF :

```
Attribut ISO "Concepts cybersécurité" :
  Identifier / Protéger / Détecter / Répondre / Rétablir
```

Cela donne un premier mapping structurel — mais il est grossier. Un contrôle ISO peut correspondre à plusieurs sous-catégories NIST, et une sous-catégorie NIST peut être partiellement couverte par plusieurs contrôles ISO.

### Étape 2 — Comparaison sémantique

Pour chaque sous-catégorie NIST, lecture de la description et des résultats attendus, puis identification des contrôles ISO qui adressent la même exigence. C'est un travail manuel — les deux référentiels utilisent des vocabulaires différents pour des choses similaires.

Exemple :

```
NIST CSF v2 — PR.AA-01 :
"Les identités et les credentials des utilisateurs, services et matériels
autorisés sont gérés par l'organisation."

Correspond à ISO 27002 :
  5.16 — Gestion des identités
  5.17 — Informations d'authentification
  8.2  — Droits d'accès privilégiés
  8.5  — Authentification sécurisée
```

### Étape 3 — Typage des relations

Toutes les correspondances ne sont pas équivalentes. Chaque relation a été typée :

| Type | Signification |
|------|---------------|
| **Complète** | Le contrôle ISO adresse entièrement l'exigence de la sous-catégorie NIST |
| **Partielle** | Le contrôle ISO adresse une partie de l'exigence |
| **Complémentaire** | Plusieurs contrôles ISO ensemble couvrent la sous-catégorie |
| **Directionnelle** | ISO → NIST ou NIST → ISO, non symétrique |

Cette distinction compte lors de l'utilisation du mapping pour l'évaluation de maturité — une correspondance partielle ne doit pas être scorée comme une correspondance complète.

---

## Points d'alignement clés

### Zones d'alignement fort

**Contrôle d'accès / Gestion des identités**
NIST PR.AA (Protéger : Gestion des identités) correspond clairement aux contrôles ISO 5.15–5.18, 8.2, 8.5. Les deux référentiels traitent le contrôle d'accès comme fondamental — la terminologie diffère mais les exigences sont quasi-identiques.

**Gestion des actifs**
NIST ID.AM (Identifier : Gestion des actifs) ↔ ISO 5.9, 5.10, 8.8. Les deux exigent un inventaire des actifs et une attribution de propriété. Les sous-catégories NIST sont légèrement plus granulaires sur les actifs logiciels et données.

**Réponse aux incidents**
NIST RS.* ↔ ISO 5.24–5.28. La fonction NIST Respond correspond bien au cluster de contrôles ISO sur la gestion des incidents. NIST est plus prescriptif sur les phases de communication et d'analyse.

### Zones de lacune

**Fonction GOVERN (nouvelle en NIST v2)**
NIST v2 ajoute une fonction entière sur la gouvernance — contexte organisationnel, rôles, politiques, risques liés à la chaîne d'approvisionnement. ISO 27002 a des contrôles équivalents (5.1–5.4, 5.19–5.22) mais ils sont dispersés sur plusieurs thèmes plutôt que regroupés comme couche de gouvernance.

**Gestion des risques**
NIST ID.RA (Évaluation des risques) est plus détaillé opérationnellement que les contrôles de traitement des risques ISO. ISO 27001 (le standard de management, pas 27002) couvre le processus de gestion des risques, ce qui crée un écart lors du mapping avec 27002 seul.

**Granularité de la détection**
La fonction NIST DE (Détecter) est plus granulaire que les contrôles de détection ISO. ISO 8.15–8.16 (journalisation, surveillance) couvrent globalement NIST DE, mais NIST distingue détection des événements, surveillance continue et amélioration des processus de détection d'une façon qu'ISO ne fait pas.

---

## Le simulateur de maturité

Sur la base du mapping, un outil a été construit pour permettre aux équipes d'estimer leur maturité NIST CSF à partir des résultats d'évaluation ISO 27002 (ou inversement).

### Fonctionnement

```
Entrée : Scores des contrôles ISO (échelle 0–4, basée sur les niveaux d'audit ISO 27001)
       ↓
Table de mapping (contrôle → relations sous-catégories avec pondérations)
       ↓
Agrégation pondérée par sous-catégorie NIST
       ↓
Sortie : Niveau de maturité NIST CSF estimé par fonction/catégorie
```

L'échelle de notation s'aligne sur les Tiers NIST CSF (1–4) et les niveaux de maturité ISO (Non implémenté → Partiellement → Largement → Entièrement implémenté).

### Logique de pondération

Une relation ISO→NIST complète contribue 100% du score ISO à la sous-catégorie. Une relation partielle contribue 50 à 75% selon le recouvrement sémantique évalué lors du mapping.

```python
# Simplifié
score_nist = sum(
    score_iso[controle] * poids
    for controle, poids in mapping[sous_categorie]
) / poids_total
```

### Exemple de sortie

```
Fonction : PROTECT
  PR.AA (Gestion identités)  — Tier estimé : 3 (Répétable)
  PR.AT (Sensibilisation)    — Tier estimé : 2 (Informing)
  PR.DS (Sécurité données)   — Tier estimé : 3 (Répétable)
  PR.PS (Sécurité platform)  — Tier estimé : 2 (Informing)
  PR.IR (Résilience)         — Tier estimé : 1 (Partiel)
```

---

## Limites

**C'est une estimation, pas un audit.** Le mapping peut indiquer approximativement où vous en êtes sur NIST au vu de votre posture ISO — il ne remplace pas une évaluation NIST CSF réelle menée par quelqu'un qui connaît l'organisation.

**Le mapping est subjectif.** La comparaison sémantique implique du jugement. Deux personnes mappant les mêmes contrôles peuvent ne pas s'accorder sur "complète" ou "partielle".

**Les référentiels ne sont pas équivalents.** Ils ont été conçus pour des contextes différents. Certaines sous-catégories NIST n'ont pas de bon équivalent ISO, et certains contrôles ISO couvrent des aspects absents du NIST.

**ISO 27002 ≠ ISO 27001.** ISO 27001 est le standard de système de management. ISO 27002 n'est que le catalogue de contrôles. Si vous n'avez que des données d'évaluation 27002, vous manquez les contrôles de processus et de SMSI qu'inclut ISO 27001.

---

## Ce que j'en retiens

**Sur les référentiels :**
- La fonction GOVERN de NIST CSF v2 est un ajout significatif — elle force les organisations à penser la cybersécurité au niveau stratégique, pas seulement opérationnel
- ISO 27002:2022 est une amélioration substantielle par rapport à la version 2013 — le tagging par attributs facilite beaucoup l'amorçage des mappings
- Aucun mapping n'est parfait ; la valeur est d'accélérer et de rendre plus cohérente la communication inter-référentiels

**Sur le travail GRC en pratique :**
- L'essentiel de l'effort est dans la comparaison sémantique — la compréhension technique compte même dans les rôles de gouvernance
- Les outils construits sur des référentiels ne valent que leur mapping sous-jacent
- La communication est le travail : le livrable technique doit être explicable aux parties prenantes non-techniques

---

## Ressources

- [NIST CSF v2 — documentation officielle](https://www.nist.gov/cyberframework)
- [NIST CSF v2 — outil de référence](https://csrc.nist.gov/projects/cybersecurity-framework)
- [ISO/IEC 27002:2022](https://www.iso.org/standard/75652.html)
- [Correspondance NIST → ISO 27001 (NIST SP 800-53)](https://csrc.nist.gov/projects/risk-management/sp800-53-controls/mappings)
- [Méthode EBIOS RM (ANSSI)](https://www.ssi.gouv.fr/guide/ebios-risk-manager-the-method/)