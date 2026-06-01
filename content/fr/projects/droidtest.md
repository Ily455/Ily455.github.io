---
title: "DroidTest"
date: 2026-03-06
draft: false
description: "Un outil CLI Python qui exécute des lots de commandes ADB de diagnostic sur des appareils Android, rapporte les résultats commande par commande et exporte en JSON — avec ciblage de device, timeouts et filtrage sélectif."
tags: ["python", "android", "adb", "cli", "testing", "automation"]
---

![Flux d'exécution DroidTest](/images/projects/droidtest/workflow.svg)

## Ce que ça fait

DroidTest est un runner en ligne de commande pour les diagnostics d'appareils Android. On lui donne une liste de sous-commandes `adb`, il les exécute toutes et produit un rapport pass/fail structuré.

```bash
python DroidTest.py -v --timeout 8 --json report.json
```

```
  _____            _     _ _______        _
 |  __ \          (_)   | |__   __|      | |
 | |  | |_ __ ___  _  __| |  | | ___  ___| |_ ___ _ __
 | |  | | '__/ _ \| |/ _` |  | |/ _ \/ __| __/ _ \ '__|
 | |__| | | | (_) | | (_| |  | |  __/\__ \ ||  __/ |
 |_____/|_|  \___/|_|\__,_|  |_|\___||___/\__\___|_|

PASS: shell getprop ro.product.model
PASS: shell dumpsys battery
FAIL: shell dumpsys proximity
PASS: shell wm size
...

Résumé : 31 passés, 3 échoués, 34 total
```

---

## Le problème résolu

Tester l'état matériel et logiciel d'un appareil Android manuellement signifie taper `adb shell ...` des dizaines de fois, lire les sorties et noter ce qui passe ou échoue. C'est fastidieux et source d'erreurs.

DroidTest automatise la boucle : définir la liste une fois, lancer, obtenir un rapport structuré. Utile pour :

- Vérifier un appareil après un flash de ROM custom
- Contrôles QA matériel (capteurs, réseau, stockage)
- Automatiser des étapes adb répétitives dans un workflow
- Obtenir un snapshot rapide de l'état d'un device connecté

---

## Architecture

Le projet a été refactorisé d'un script monolithique vers un package structuré :

```
DroidTest.py          ← point d'entrée (rétrocompatible)
droidtest/
  __init__.py
  core.py             ← chargement des commandes et logique d'exécution
  cli.py              ← parsing des arguments, affichage, bannière
list.txt              ← liste de commandes par défaut
tests/
  test_core.py        ← tests unitaires
```

Cette séparation permet de tester la logique principale (chargement des commandes, exécution adb, collecte des résultats) indépendamment du comportement CLI.

---

## Fonctionnement

### Chargement des commandes (`core.py`)

Les commandes sont lues depuis un fichier texte. Chaque ligne est une sous-commande `adb` — le préfixe `adb` est ajouté automatiquement. Les commentaires (`#`) et les lignes vides sont ignorés.

```text
# Commentaire — ignoré
shell getprop ro.product.model
shell dumpsys battery
shell wm size
```

```python
def load_commands(file_path):
    for line in path.read_text().splitlines():
        line = line.strip()
        if not line or line.startswith("#"):
            continue
        commands.append(line)
```

### Exécution des commandes

Chaque commande est lancée via `subprocess.run` avec `shlex.split` pour une gestion sûre des arguments. Les résultats sont capturés dans des instances du dataclass `CommandResult` avec `command`, `returncode`, `stdout`, `stderr` et une propriété `ok`.

```python
completed = subprocess.run(
    ["adb"] + shlex.split(command),
    capture_output=True, text=True, timeout=timeout
)
```

`shlex.split` est important — il gère correctement les commandes avec des arguments entre guillemets, là où un simple `.split()` casserait les espaces à l'intérieur des arguments.

### Ciblage de device

Si plusieurs appareils sont connectés, utiliser `--device` pour en cibler un :

```bash
adb devices
# List of devices attached
# R3CN90XXXXX    device
# emulator-5554  device

python DroidTest.py --device R3CN90XXXXX
```

Cela passe `-s <serial>` à chaque invocation `adb`.

---

## Bibliothèque de commandes intégrée

Le fichier `OtherUsefulTests.txt` contient une liste de référence de plus de 60 commandes adb documentées, couvrant :

- Écran tactile, capteurs (accéléromètre, gyroscope, baromètre, GPS, NFC, proximité, boussole)
- Santé de la batterie, température, tension, capacité
- Wi-Fi, Bluetooth, données mobiles, signal réseau
- Vitesse lecture/écriture stockage (benchmark `dd`)
- Caméra, microphone, haut-parleur, lampe de poche, vibration
- Fréquence et température CPU
- RAM, résolution et densité d'écran
- Sécurité : statut root, état du chiffrement, niveau de patch de sécurité
- Infos appareil : modèle, fabricant, numéro de série, IMEI, version Android, noyau

---

## Options CLI

| Flag | Description |
|------|-------------|
| `-c, --commands-file` | Chemin vers la liste de commandes (défaut : `list.txt`) |
| `-d, --device` | Numéro de série du device ADB (`adb -s ...`) |
| `-t, --timeout` | Timeout par commande en secondes |
| `-v, --verbose` | Affiche stdout/stderr par commande |
| `-S, --success-only` | Affiche uniquement les commandes qui passent |
| `-F, --fail-only` | Affiche uniquement les commandes qui échouent |
| `--stop-on-failure` | Arrête après le premier échec |
| `--json FILE` | Écrit les résultats complets en JSON |
| `--success-file FILE` | Ajoute les détails des succès dans un fichier |
| `--fail-file FILE` | Ajoute les détails des échecs dans un fichier |

---

## Format du rapport JSON

```json
{
  "summary": {
    "passed": 31,
    "failed": 3,
    "total": 34
  },
  "results": [
    {
      "command": "shell getprop ro.product.model",
      "returncode": 0,
      "stdout": "Pixel 6",
      "stderr": "",
      "ok": true
    },
    {
      "command": "shell dumpsys proximity",
      "returncode": 1,
      "stdout": "",
      "stderr": "No proximity sensor found",
      "ok": false
    }
  ]
}
```

---

## Avant / Après le refactoring

Le `backup.py` original était un script monolithique : parsing des arguments, exécution et affichage dans une seule fonction, avec des listes de commandes codées en dur.

| Aspect | Avant | Après |
|--------|-------|-------|
| Structure | Script plat unique | Package avec `core.py` + `cli.py` |
| Chargement des commandes | Liste codée en dur | Fichier externe, parser commentaires |
| Parsing des arguments | `.split()` sur strings | `shlex.split()` — gère les args entre guillemets |
| Gestion du timeout | Aucune | Timeout par commande avec `subprocess` |
| Ciblage du device | Aucun | Flag `--device` → `adb -s` |
| Filtrage de sortie | Aucun | `--success-only`, `--fail-only` |
| Export rapport | Aucun | JSON, fichiers log succès/échec |
| Codes de sortie | Toujours 0 | `0` tout passe, `2` au moins un échec |
| Tests | Aucun | Tests unitaires dans `tests/` |

---

## Connecter un appareil

1. **Activer les options développeur** — Paramètres → À propos du téléphone → taper 7 fois sur Numéro de build
2. **Activer le débogage USB** — Paramètres → Options développeur → Débogage USB
3. Connecter en USB, accepter la clé RSA sur le téléphone
4. Vérifier : `adb devices` doit afficher le device avec le statut `device`

Si le device s'affiche `unauthorized`, révoquer les autorisations de débogage USB sur le device et reconnecter. Si rien n'apparaît : `adb kill-server && adb start-server`.

---

## Lancer les tests

```bash
python -m unittest discover -s tests -p 'test_*.py'
```

---

## Ressources

- [Documentation Android ADB](https://developer.android.com/tools/adb)
- [Référence des commandes adb shell](https://developer.android.com/tools/adb#shellcommands)
- [Module Python subprocess](https://docs.python.org/3/library/subprocess.html)
- [Module Python shlex](https://docs.python.org/3/library/shlex.html)
