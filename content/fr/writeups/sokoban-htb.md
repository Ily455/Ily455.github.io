---
title: "SokobanHTB — Reverse d'un jeu SFML pour retrouver un flag chiffré en TEA"
date: 2026-08-10
draft: false
description: "Challenge HTB reversing / game-pwn : un binaire Windows SFML façon Sokoban avec un plateau volontairement insoluble. Analyse statique Ghidra, reconstitution de la condition de victoire, dérivation de la clé depuis les coordonnées pixel des cibles X, et déchiffrement TEA du flag."
tags: ["reverse-engineering", "ghidra", "tea", "cryptography", "ctf", "htb"]
---

> Challenge HTB reversing / game-pwn. Le jeu est un binaire Windows SFML qui affiche un puzzle Sokoban — mais une caisse est placée *hors* des murs, rendant le niveau insoluble en jouant. C'est l'indice : il ne faut pas y jouer, il faut le reverser.

---

![SokobanHTB — du plateau à la dérivation de la clé TEA](/images/writeups/sokoban/diagram.svg)

## Reconnaissance

```
file SokobanHTB.exe
→ PE32+ executable (console) x86-64, for MS Windows
```

`strings` ne renvoie quasiment que du bruit SFML / OpenGL / FreeType. Les seules chaînes notables sont `Sokoban HTB`, `arial.ttf` et les chemins d'assets `assets\player.png`, `assets\X.png`, `assets\box.png`. Les assets sont chargés depuis le disque et le flag n'est **pas** en clair — il est donc calculé à l'exécution, sur la condition de victoire.

---

## Analyse statique dans Ghidra

Chargé à `ImageBase 0x140000000`. La référence croisée sur la chaîne `Sokoban HTB` mène directement à la fonction principale `FUN_1400045f0`, qui :

- initialise la fenêtre SFML et charge les textures,
- construit une **grille 7 × 8** à partir d'un tableau d'entiers local (`local_8b8[]`),
- exécute une double boucle qui instancie les sprites selon les valeurs de la grille,
- entre dans la boucle de jeu,
- et, à la victoire (`if (bVar5)`), exécute un **bloc de déchiffrement du flag**.

Valeurs de la grille : `0 = vide`, `1 = mur`, `2 = cible (X)`, `3 = caisse`, `4 = joueur`.

```
Row 0:  .  #  #  #  #  .  .
Row 1:  #  #  P  X  #  .  .
Row 2:  #  .  B  .  #  .  .
Row 3:  #  .  B  .  #  #  #
Row 4:  #  .  B  #  .  .  #
Row 5:  #  X  .  .  .  X  #
Row 6:  #  #  .  .  #  #  #
Row 7:  .  #  #  #  #  .  .
```

Le rendu des sprites utilise un mapping fixe — case de 64 px, décalage (320, 90) :

```
x = col * 64 + 320
y = row * 64 + 90
```

### La caisse hors-cadre

Juste après la boucle d'init, un appel codé en dur repositionne une caisse :

```
FUN_140018fb0(box_sprite + 0x120, 0x43cd000044800000)   ; @ 0x140004dce
→ setPosition(x = 1024.0, y = 410.0)
```

`x = 1024` est hors du plateau (la colonne la plus à droite, col 6, est à `6*64+320 = 704`). C'est ce qui rend le puzzle insoluble à la main — et confirme que le chemin voulu est le reverse de la logique de victoire, pas le jeu.

---

## Condition de victoire et dérivation de la clé

Le test de victoire est un simple `TEST AL, AL ; JE` à `0x140005513` : si toutes les caisses sont sur leurs cibles, le bloc flag s'exécute.

Dans ce bloc, la **clé** est construite à partir des positions des caisses *au moment de la victoire* — c'est-à-dire les coordonnées des cibles X :

```
key[0] = (int) y de X1   = 154   (cible col 3, row 1 → y = 1*64+90)
key[1] = (int) x de X1   = 512   (col 3 → x = 3*64+320)
key[2] = (int) x de X2   = 384   (cible col 1, row 5 → x = 1*64+320)
key[3] = (int) x de X3   = 640   (cible col 5, row 5 → x = 5*64+320)

key = [154, 512, 384, 640]
```

L'ordre exact des deux dernières valeurs de la clé (les X de col 1 et col 5) a été confirmé en testant les deux permutations : une seule donnait un déchiffrement lisible.

Le **texte chiffré** (40 octets = 5 blocs de 8) est copié depuis la pile :

```
ea0109d5 c8ee03d2   553f3fc8 a8b34cd9   3631ad3e 98689219
4e883f78 082e3c84   9a6a6cc0 5ab5c6a4
```

---

## Le chiffrement — TEA

`FUN_140003f00` est le chiffreur. Le décompilateur de Ghidra le déroule en quelque chose qui paraît custom, mais les constantes le trahissent : **c'est du TEA standard** (Tiny Encryption Algorithm).

- 32 rounds
- `delta = 0x9E3779B9` (la constante du nombre d'or — une signature de TEA)
- somme initiale `= delta * 32 = 0xC6EF3720`

Reconnaître `0x9E3779B9` est toute l'astuce : une fois qu'on sait que c'est du TEA, la routine de déchiffrement est classique.

---

## Déchiffrement

```python
import struct

def tea_decrypt(v0, v1, key):
    MASK = 0xFFFFFFFF
    delta = 0x9E3779B9
    s = (delta * 32) & MASK          # 0xC6EF3720
    k0, k1, k2, k3 = key
    for _ in range(32):
        v1 = (v1 - (((v0 << 4) + k2) ^ (v0 + s) ^ ((v0 >> 5) + k3))) & MASK
        v0 = (v0 - (((v1 << 4) + k0) ^ (v1 + s) ^ ((v1 >> 5) + k1))) & MASK
        s  = (s - delta) & MASK
    return v0, v1

ct = [0xea0109d5, 0xc8ee03d2, 0x553f3fc8, 0xa8b34cd9, 0x3631ad3e,
      0x98689219, 0x4e883f78, 0x082e3c84, 0x9a6a6cc0, 0x5ab5c6a4]

key = [154, 512, 384, 640]

pt = b""
for i in range(0, len(ct), 2):
    v0, v1 = tea_decrypt(ct[i], ct[i+1], key)
    pt += struct.pack("<II", v0, v1)

print(pt.rstrip(b"\x00").decode())   # → le flag
```

---

## Le bug qui a coûté des heures

Le premier dword du chiffré avait été mal lu depuis Ghidra :

```
lu (faux)  : 0xea010dd5    (0x0d)
correct    : 0xea0109d5    (0x09)
```

Ce seul quartet erroné faisait déchiffrer le bloc 0 en charabia alors que les blocs 1 à 4 étaient corrects. Ça m'a fait chercher une mauvaise clé et un mauvais chiffreur pendant des heures — tout le reste (clé, TEA, grille) était juste depuis le début.

**Leçon :** toujours revérifier les octets codés en dur lus dans un décompilateur, surtout les critiques. Quand un seul bloc échoue mais que les autres déchiffrent proprement, soupçonner la donnée avant l'algorithme.

---

## Ce que j'en retire

- Reverser un jeu SFML : localiser la boucle de jeu et la condition de victoire à partir d'une chaîne d'UI, puis lire la grille directement dans le tableau d'initialisation.
- La géométrie à l'écran peut être le matériau de la clé — ici la clé était littéralement les coordonnées pixel des cibles (`col*64+320`, `row*64+90`).
- Reconnaître TEA à son delta `0x9E3779B9`, même quand le décompilateur déroule les rounds en code méconnaissable.
- La rigueur de transcription : une erreur de lecture d'un quartet est indiscernable d'un bug d'algorithme tant qu'on ne l'isole pas bloc par bloc.

---

## Adresses Ghidra

```
0x1400045f0  boucle de jeu principale
0x140003f00  chiffreur TEA
0x140018fb0  setPosition SFML
0x140004dce  CALL setPosition de la caisse hors-cadre (RDX = 0x43cd000044800000)
0x140005513  condition de victoire (TEST AL,AL ; JE)
0x140005a13  memmove du flag déchiffré → std::string
0x140022800  affichage du flag

ImageBase = 0x140000000   →   RVA = addr - 0x140000000
```

---

## Ressources

- [Ghidra](https://ghidra-sre.org/)
- [TEA — Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm)
- [SFML](https://www.sfml-dev.org/)
