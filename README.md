# ily455.github.io

Personal portfolio and technical blog — cybersecurity projects, writeups, and research.

**Live at:** [ily455.github.io](https://ily455.github.io)

## Stack

- [Hugo](https://gohugo.io/) static site generator
- [hugo-noir](https://github.com/Ily455/hugo-noir) theme (forked)
- GitHub Pages + GitHub Actions (auto-deploy on push to `main`)
- Bilingual EN / FR

## Structure

```
content/
  en/projects/    # project write-ups
  en/writeups/    # CTF, lab, and research write-ups
  fr/             # French translations
static/images/    # SVG diagrams and card images
layouts/          # theme overrides
```

## Local dev

```bash
git clone --recurse-submodules https://github.com/Ily455/Ily455.github.io
hugo server -D
```

Requires Hugo extended v0.112+.
