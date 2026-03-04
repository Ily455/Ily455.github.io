# Briefing — Rebuild of ily455.github.io

## Context
You are helping rebuild a personal blog from scratch for Ilyass El Annid, a cybersecurity engineer based in Marseille, France. The blog is hosted on GitHub Pages at ily455.github.io. The old Jekyll site has been wiped from the local folder — only the `.git` folder remains. The goal is to set up a clean Hugo site with the xterm theme and deploy it via GitHub Actions.

## Stack
- **Static site generator**: Hugo
- **Theme**: hugo-xterm (https://github.com/manid2/hugo-xterm)
- **Hosting**: GitHub Pages (ily455.github.io)
- **Deployment**: GitHub Actions

## Site structure
Three sections only, nothing else:
- **About** — who Ilyass is, his background, his interests
- **Write-ups** — technical write-ups (CTF challenges, security research notes)
- **Projects** — documented personal projects with context and learnings

## About page content
Use this to write the About page. Keep it factual, no hype, no overselling:

Ilyass is a cybersecurity engineer who recently graduated with an MSc in IT Reliability & Security from Aix-Marseille University and a State Engineering Degree in Cybersecurity from ENSA Oujda. He has interned at Secure-IC (R&D, code obfuscation on RISC-V/ARM, LLVM, FPGA), Enedis (cybersecurity governance and compliance), and Techso Group (malware analysis). He is interested in low-level security, reverse engineering, embedded systems security, and vulnerability research. He likes working close to the hardware, reading papers, and understanding how things work at the lowest level.

Contact:
- Email: ilyass.elannid@gmail.com
- GitHub: github.com/Ily455
- LinkedIn: linkedin.com/in/ilyass-elannid

## Projects to document
Create placeholder project pages for these (no fake content, just structure with real descriptions):

1. **Py-C-obfuscator** — C source-to-source obfuscator written in Python
2. **IM-LLVM-Pass** — LLVM pass for identifier mangling/renaming (randomizes function names, global variables, structure names)
3. **FSParser** — FAT32 & EXT filesystem parser in Python
4. **Android Fuzzing Study** — Study and implementation of fuzzing on Android (generational and mutational fuzzing, AFL, two-VM environment Linux+Android via ADB/TCP, attempted to reproduce a known multimedia vulnerability, discovered real-world infrastructure limitations)

## Write-ups
Leave the write-ups section empty for now — just the section structure with a placeholder message like "Write-ups coming soon."

## Deployment
Set up GitHub Actions to automatically build and deploy to GitHub Pages on every push to main. Use the standard Hugo GitHub Actions workflow.

## Important constraints
- Keep the design clean and minimal — xterm theme defaults are fine, do not over-customize
- English only
- No unnecessary sections, no tags cloud, no social widgets beyond what's in the About page
- The site must work on ily455.github.io — set baseURL accordingly in hugo.toml

## What to deliver
1. A fully working Hugo site in the current directory (which already has .git initialized)
2. GitHub Actions workflow file at .github/workflows/deploy.yml
3. About page written and ready
4. Project placeholder pages created
5. Empty write-ups section
6. Confirm everything builds with `hugo` before finishing
