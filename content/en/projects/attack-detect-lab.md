---
title: "ATT&CK Detection Lab"
date: 2026-05-31
draft: false
description: "Local cybersecurity lab for simulating MITRE ATT&CK techniques and detecting them with Elastic SIEM — everything in Docker, no VMs, cleanup in one command."
tags: ["elastic", "siem", "docker", "mitre-attack", "atomic-red-team", "detection", "blue-team", "sigma", "linux"]
---

> Personal lab project, 2026 — **work in progress.** Goal: build a self-contained environment for simulating adversary techniques and engineering the detections — one technique at a time, with logs, Sigma rules, and writeups as output.

---

## Overview

The lab runs Atomic Red Team techniques against a Linux target container and detects them with Elastic SIEM. The full pipeline — attack simulation, log collection, detection — runs locally on an Apple Silicon Mac with no VMs and no cloud dependency.

Each technique produces:
- Raw logs captured from the macOS unified logging system
- A Sigma detection rule
- A KQL query for Kibana
- A writeup documenting the technique, the evidence, and the detection logic

---

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  macOS Host (M3)                                            │
│                                                             │
│  run-test.sh                                                │
│    └── pwsh → Invoke-AtomicTest ──► SSH → target container  │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐   │
│  │  Docker  (bridge network: elastic)                   │   │
│  │                                                      │   │
│  │  target ──► Filebeat ──► Elasticsearch ◄── Kibana   │   │
│  │  (Ubuntu)   watches       stores +         SIEM UI  │   │
│  │             /logs/        indexes                    │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

**Key design decision:** Atomic Red Team runs inside the target container over SSH — not on the Mac host. The Mac only initiates the test. This means PowerShell and the Atomic framework leave zero trace on the host after cleanup, beyond Docker Desktop itself.

---

## Stack

| Component | Role | Version |
|---|---|---|
| Elasticsearch | Log storage and indexing | 8.14.3 |
| Kibana | SIEM UI, detection rules, KQL | 8.14.3 |
| Filebeat | Log shipper | 8.14.3 |
| Target container | Ubuntu ARM64, attack surface | — |
| Atomic Red Team | MITRE ATT&CK technique simulator | latest |
| PowerShell | Atomic runtime (SSH transport) | latest |

All Elastic images run as `linux/arm64` — native M3 execution, no emulation.

---

## How a Test Run Works

```bash
./run-test.sh T1059.004
```

1. Opens an SSH session from PowerShell to the target container
2. Runs `Invoke-AtomicTest T1059.004` inside the container
3. Waits for post-execution events to settle
4. Runs `Invoke-AtomicTest T1059.004 -Cleanup` to remove artifacts
5. Filebeat ships the container logs to Elasticsearch within seconds
6. Events appear in Kibana under the `attack-detect-logs-*` data view

---

## Techniques

| # | ID | Name | Status |
|---|---|---|---|
| 1 | T1059.004 | Unix Shell | Planned |
| 2 | T1053.003 | Cron Job Persistence | Planned |
| 3 | T1136.001 | Create Local Account | Planned |
| 4 | T1087.001 | Local Account Discovery | Planned |
| 5 | T1083 | File and Directory Discovery | Planned |
| 6 | T1105 | Ingress Tool Transfer | Planned |

---

## Detection Output Format

Each completed technique produces a Sigma rule targeting the `attack-detect-logs-*` index:

```yaml
title: Unix Shell Execution via bash
status: experimental
logsource:
    category: process_creation
    product: linux
detection:
    selection:
        Image|endswith: '/bash'
        CommandLine|contains: '-c'
    condition: selection
```

And a KQL query for Kibana:

```
process.name: "bash" AND process.args: "-c"
```

---

## Why Docker Over VMs

VMs add a layer that's expensive and slow to iterate on. Docker gives:
- Full stack up in under 2 minutes
- `docker compose down -v` wipes everything — no leftover state between tests
- ARM64-native images on M3, no Rosetta overhead
- The target container is a real Linux environment, not a Mac process — techniques behave the same as on a real Linux host

The tradeoff: macOS `log stream` and the unified logging system are proprietary and less structured than Linux auditd or Windows Event Log. The lab is designed with a v2 architecture in mind that moves everything into containers for a proper Linux-to-Linux setup.

---

## Roadmap — v2

Move Atomic Red Team inside a dedicated attacker container. Full SSH-based lateral movement between containers, no host involvement. Leaves the Mac completely clean.

```
Mac (SSH client only)
  └── SSH → Attacker container (PowerShell + Atomic)
                └── SSH → Target container (Ubuntu + Filebeat)
                              └── Filebeat → Elasticsearch → Kibana
```
