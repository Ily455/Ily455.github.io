---
title: "ATT&CK Detection Lab"
date: 2026-05-31
draft: false
description: "Local cybersecurity lab for simulating MITRE ATT&CK techniques and detecting them with Elastic SIEM — Atomic Red Team against a Linux target, logs shipped to Kibana."
tags: ["elastic", "siem", "docker", "mitre-attack", "atomic-red-team", "detection", "blue-team", "linux", "auditd"]
---

> Personal lab project, 2026 — **work in progress.** Goal: build a self-contained environment for simulating adversary techniques and engineering the detections — one technique at a time, with logs, Sigma rules, and writeups as output.

---

## Overview

The lab runs Atomic Red Team techniques against a dedicated Linux target and detects them with Elastic SIEM. The SIEM stack runs in Docker on an Apple Silicon Mac. Techniques execute on a separate Linux VM (UTM), which ships logs to Elasticsearch via Filebeat.

Each completed technique produces:
- Raw logs from auditd and syslog on the Linux target
- A Sigma detection rule
- A KQL query for Kibana
- A writeup documenting the technique, the evidence, and the detection logic

---

## Architecture

```
┌──────────────────────────────────────────────────────────────┐
│  macOS Host (M3)                                             │
│                                                              │
│  run-test.sh                                                 │
│    └── SSH → Linux VM (UTM)                                  │
│               ├── Atomic Red Team executes techniques        │
│               ├── auditd records every execve syscall        │
│               └── Filebeat ──► Elasticsearch ◄── Kibana      │
│                                (Docker)         (Docker)     │
└──────────────────────────────────────────────────────────────┘
```

**Key design decision:** Atomic Red Team and all attack simulation stay on the Linux target — not on the Mac. The Mac only initiates tests over SSH. The SIEM (Elasticsearch + Kibana) runs in Docker and never touches the target directly — it just receives logs.

---

## Stack

| Component | Role | Version |
|---|---|---|
| Elasticsearch | Log storage and indexing | 8.14.3 |
| Kibana | SIEM UI, detection rules, KQL | 8.14.3 |
| Filebeat | Log shipper (runs on target VM) | 8.14.3 |
| Target VM | Ubuntu 22.04 ARM64, attack surface | UTM on M3 |
| Atomic Red Team | MITRE ATT&CK technique simulator | latest |
| PowerShell | Atomic runtime | 7.4.6 |

Elastic images run as `linux/arm64` — native M3 execution, no emulation.

---

## How a Test Run Works

```bash
./run-test.sh T1059.004
```

1. SSH into the Linux target VM
2. Run `Invoke-AtomicTest T1059.004` on the target
3. auditd records every `execve` syscall — every command that ran, with arguments
4. Filebeat ships auditd logs and syslog to Elasticsearch within seconds
5. Events appear in Kibana under the `attack-detect-logs-*` data view
6. `Invoke-AtomicTest T1059.004 -Cleanup` removes artifacts

---

## Techniques

| # | ID | Name | Status |
|---|---|---|---|
| 1 | T1059.004 | Unix Shell | Done |
| 2 | T1053.003 | Cron Job Persistence | Done |
| 3 | T1136.001 | Create Local Account | Done |
| 4 | T1087.001 | Local Account Discovery | Done |
| 5 | T1083 | File and Directory Discovery | Done |
| 6 | T1105 | Ingress Tool Transfer | Done |

### T1059.004 — Unix Shell

15/17 sub-tests executed successfully. auditd captured every shell invocation via `execve`.

Sample log evidence:
```
type=EXECVE msg=audit(1780489845.066:1912): argc=3 a0="sh" a1="-c" a2=""
```

**KQL detection:**
```
message: "type=EXECVE" AND (message: "a0=\"sh\"" OR message: "a0=\"bash\"" OR message: "a0=\"dash\"")
```

---

### T1053.003 — Cron Job Persistence

4/4 sub-tests executed successfully. auditd captured writes to cron directories and bash commands via PROCTITLE records.

Key evidence — crontab loaded from /tmp/:
```
type=EXECVE argc=2 a0="crontab" a1="/tmp/persistevil"
```

**KQL detection:**
```
message: "type=EXECVE" AND message: "a0=\"crontab\""
```

---

### T1136.001 — Create Local Account

Atomic created a local user named `evil_user`. auditd captured both creation and deletion via `useradd`/`userdel`.

Key evidence — PROCTITLE hex decoded:
```
userdel evil_user
```

Notable: `auid=1000` persists across sudo — auditd tracks the original user (`ubuntu`) even when the command runs as root.

**KQL detection:**
```
message: "type=EXECVE" AND (message: "a0=\"useradd\"" OR message: "a0=\"adduser\"")
```

---

### T1087.001 — Local Account Discovery

Atomic ran enumeration commands: `cat /etc/passwd`, `cat /etc/shadow`, `lsof`, `getent passwd`. auditd captured every invocation.

```
type=EXECVE argc=2 a0="cat" a1="/etc/passwd"
type=EXECVE argc=1 a0="lsof"
```

**KQL detection:**
```
message: "type=EXECVE" AND (message: "a1=\"/etc/passwd\"" OR message: "a1=\"/etc/shadow\"" OR message: "a0=\"lsof\"")
```

---

### T1083 — File and Directory Discovery

3/3 tests succeeded: file discovery via `ls /tmp`, recursive directory tree enumeration, and `showmount` for network share discovery.

```
type=EXECVE argc=2 a0="find" a1="/"
type=EXECVE argc=1 a0="showmount"
```

**KQL detection:**
```
message: "type=EXECVE" AND (message: "a0=\"find\"" OR message: "a0=\"showmount\"" OR message: "a0=\"tree\"")
```

---

### T1105 — Ingress Tool Transfer

6/9 tests succeeded. A dedicated `remote` container (Ubuntu + SSH + rsync) was added to enable the transfer tests. rsync push/pull, scp push, sftp push, and curl download+execute all passed.

```
type=EXECVE a0="curl" a1="-s" a2="<url>"
```

**KQL detection:**
```
message: "type=EXECVE" AND (message: "a0=\"curl\"" OR message: "a0=\"wget\"")
```

---

## Engineering Constraints Found During Build

The first version of this lab used a Docker container as the target instead of a VM. This uncovered several constraints worth documenting.

**Docker Desktop's linuxkit kernel has no audit subsystem.** Running Ubuntu in a Docker container on macOS means the container shares Docker Desktop's internal Linux VM — not a standard kernel. The audit subsystem (`auditd`, `CAP_AUDIT_*`) is not compiled in. `auditctl -s` returns "Operation not permitted" regardless of capabilities granted. There is no workaround short of replacing the kernel.

**PowerShell script block logging on Linux requires systemd/journald.** Enabling `ScriptBlockLogging` in `powershell.config.json` silently does nothing in a minimal container — there is no journald to write to. The logs disappear.

**Consequence:** A container target can only capture what it deliberately writes to known log files — auth events, cron events, explicit output redirects. It cannot passively record what commands ran. That is not real SIEM monitoring.

**Solution:** Move the target to a proper Linux VM (UTM, Ubuntu 22.04). The VM has a real kernel, real systemd, real auditd. Two audit rules capture every `execve` call on the system — every command that runs, regardless of how the attacker triggered it.

```bash
auditctl -a always,exit -F arch=b64 -S execve -k exec_commands
auditctl -a always,exit -F arch=b32 -S execve -k exec_commands
```

**ARM64 note:** Microsoft's Linux package repository does not publish PowerShell for ARM64. Installation requires downloading the tarball directly from GitHub releases.

---

## Roadmap

- ~~Complete UTM VM setup with auditd + Filebeat~~ ✓
- ~~T1059.004 — Unix Shell~~ ✓
- ~~T1053.003 — Cron Job Persistence~~ ✓
- ~~T1136.001 — Create Local Account~~ ✓
- ~~T1087.001 — Local Account Discovery~~ ✓
- ~~T1083 — File and Directory Discovery~~ ✓
- ~~T1105 — Ingress Tool Transfer~~ ✓
