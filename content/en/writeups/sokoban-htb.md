---
title: "SokobanHTB — Reversing an SFML Game to Recover a TEA-Encrypted Flag"
date: 2026-08-10
draft: false
description: "HTB game-pwn / reversing challenge: a Windows SFML Sokoban binary with a deliberately unsolvable board. Ghidra static analysis, reconstruction of the win condition, key derivation from on-screen X-mark pixel coordinates, and TEA decryption of the flag."
tags: ["reverse-engineering", "ghidra", "tea", "cryptography", "ctf", "htb"]
---

> HTB reversing / game-pwn challenge. The game ships as a Windows SFML binary that renders a Sokoban puzzle — but one box is placed *outside* the walls, so it can never be solved by playing. That is the hint: don't play it, reverse it.

---

![SokobanHTB — board layout to TEA key derivation](/images/writeups/sokoban/diagram.svg)

## Recon

```
file SokobanHTB.exe
→ PE32+ executable (console) x86-64, for MS Windows
```

`strings` returns mostly SFML / OpenGL / FreeType noise. The only notable strings are `Sokoban HTB`, `arial.ttf`, and the asset paths `assets\player.png`, `assets\X.png`, `assets\box.png`. The assets are loaded from disk and the flag is **not** present in cleartext — so the flag is computed at runtime, on the win condition.

---

## Static analysis in Ghidra

Loaded at `ImageBase 0x140000000`. Cross-referencing the `Sokoban HTB` string leads straight to the main function `FUN_1400045f0`, which:

- initializes the SFML window and loads the textures,
- builds a **7 × 8 grid** from a local integer array (`local_8b8[]`),
- runs a double loop that instantiates sprites from the grid values,
- enters the game loop,
- and, on victory (`if (bVar5)`), runs a **flag-decryption block**.

Grid values: `0 = empty`, `1 = wall`, `2 = target (X)`, `3 = box`, `4 = player`.

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

Sprite rendering uses a fixed mapping — tile size 64 px, offset (320, 90):

```
x = col * 64 + 320
y = row * 64 + 90
```

### The out-of-bounds box

Right after the init loop, a hardcoded call repositions one box sprite:

```
FUN_140018fb0(box_sprite + 0x120, 0x43cd000044800000)   ; @ 0x140004dce
→ setPosition(x = 1024.0, y = 410.0)
```

`x = 1024` is off the board (the rightmost column, col 6, sits at `6*64+320 = 704`). That is what makes the puzzle unsolvable by hand — and confirms the intended path is reversing the win logic, not playing.

---

## Win condition and key derivation

The win check is a simple `TEST AL, AL ; JE` at `0x140005513`: if all boxes sit on their targets, the flag block runs.

Inside that block, the **key** is built from the box positions *at the moment of victory* — i.e. the coordinates of the X targets:

```
key[0] = (int) y of X1   = 154   (target at col 3, row 1 → y = 1*64+90)
key[1] = (int) x of X1   = 512   (col 3 → x = 3*64+320)
key[2] = (int) x of X2   = 384   (target at col 1, row 5 → x = 1*64+320)
key[3] = (int) x of X3   = 640   (target at col 5, row 5 → x = 5*64+320)

key = [154, 512, 384, 640]
```

The exact order of the last two key values (the col-1 and col-5 X marks) was confirmed by testing both permutations — only one produced readable plaintext.

The **ciphertext** (40 bytes = 5 blocks of 8) is copied from the stack:

```
ea0109d5 c8ee03d2   553f3fc8 a8b34cd9   3631ad3e 98689219
4e883f78 082e3c84   9a6a6cc0 5ab5c6a4
```

---

## The cipher — TEA

`FUN_140003f00` is the cipher. Ghidra's decompiler unrolls it into something that looks custom, but the constants give it away: **it is standard TEA** (Tiny Encryption Algorithm).

- 32 rounds
- `delta = 0x9E3779B9` (the golden-ratio constant — a TEA fingerprint)
- initial sum `= delta * 32 = 0xC6EF3720`

Recognizing `0x9E3779B9` is the whole trick: once you know it's TEA, the decrypt routine is textbook.

---

## Decryption

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

print(pt.rstrip(b"\x00").decode())   # → the flag
```

---

## The bug that cost hours

The first ciphertext dword was misread from Ghidra:

```
misread : 0xea010dd5    (0x0d)
correct : 0xea0109d5    (0x09)
```

That single wrong nibble made block 0 decrypt to garbage while blocks 1–4 were fine. It sent me chasing a wrong key and a wrong cipher for hours — everything else (key, TEA, grid) had been correct from the start.

**Lesson:** always double-check hardcoded bytes read from a decompiler, especially the critical ones. When a single block fails but the rest decrypt cleanly, suspect the data before the algorithm.

---

## What I learned

- Reversing an SFML game: locate the game loop and win condition from a UI string, then read the grid straight out of the initialization array.
- On-screen geometry can be the key material — the flag key was literally the pixel coordinates of the targets (`col*64+320`, `row*64+90`).
- Recognizing TEA by its `0x9E3779B9` delta, even when the decompiler unrolls the rounds into unfamiliar-looking code.
- Disciplined data transcription: a one-nibble read error is indistinguishable from an algorithm bug until you isolate it block by block.

---

## Ghidra addresses

```
0x1400045f0  main game loop
0x140003f00  TEA cipher
0x140018fb0  SFML setPosition
0x140004dce  CALL setPosition of the off-grid box (RDX = 0x43cd000044800000)
0x140005513  win condition  (TEST AL,AL ; JE)
0x140005a13  memmove of decrypted flag → std::string
0x140022800  flag display

ImageBase = 0x140000000   →   RVA = addr - 0x140000000
```

---

## Resources

- [Ghidra](https://ghidra-sre.org/)
- [TEA — Tiny Encryption Algorithm](https://en.wikipedia.org/wiki/Tiny_Encryption_Algorithm)
- [SFML](https://www.sfml-dev.org/)
