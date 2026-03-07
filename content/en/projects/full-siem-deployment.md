---
title: "Full SIEM Deployment"
date: 2026-03-06
draft: false
description: "End-to-end deployment of a Security Information and Event Management stack — log collection, normalization, correlation rules, dashboards, and alerting — built for a simulated enterprise environment."
tags: ["siem", "splunk", "zabbix", "security-monitoring", "log-management", "blue-team"]
---

## Overview

This project was a full deployment of a SIEM (Security Information and Event Management) system from scratch — log sources, collection agents, normalization, correlation rules, and dashboards — in a simulated enterprise environment.

The goal was practical: understand every layer of a SIEM, not just use a pre-configured one. That means knowing why a log was normalized a certain way, why a correlation rule fires (or doesn't), and what a realistic alert looks like versus noise.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  Log Sources                                            │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐  │
│  │ Windows  │ │  Linux   │ │ Network  │ │  Web     │  │
│  │ Event Log│ │  syslog  │ │ devices  │ │  server  │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └────┬─────┘  │
└───────┼─────────────┼─────────────┼─────────────┼───────┘
        │             │             │             │
        ▼             ▼             ▼             ▼
┌──────────────────────────────────────────────────────┐
│  Collection layer (Splunk Universal Forwarder / syslog-ng) │
└─────────────────────────┬────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│  SIEM Core (Splunk / ELK)                            │
│  ├── Indexer (storage + search)                      │
│  ├── Search head (queries + dashboards)              │
│  └── Correlation engine (alert rules)                │
└─────────────────────────┬────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────┐
│  Monitoring & Alerting                               │
│  ├── Zabbix (infrastructure health)                  │
│  └── Alert notifications                             │
└──────────────────────────────────────────────────────┘
```

---

## Log Sources

### Windows event logs

Windows machines send events via Splunk Universal Forwarder. Key event IDs monitored:

| Event ID | Meaning |
|----------|---------|
| 4624 | Successful logon |
| 4625 | Failed logon attempt |
| 4648 | Logon with explicit credentials |
| 4688 | New process created |
| 4698/4702 | Scheduled task created/modified |
| 4720 | User account created |
| 4732 | Member added to privileged group |
| 7045 | New service installed |

### Linux syslog

Linux hosts send `/var/log/auth.log`, `/var/log/syslog`, and application logs via syslog-ng to the SIEM indexer. Key patterns:

```
Failed password for ... from ...   ← brute force indicator
Accepted publickey for ...         ← SSH login
sudo: ... command not allowed      ← privilege escalation attempt
```

### Network devices

Firewall and switch logs forwarded via syslog. Key events: connection allowed/denied, policy violations, interface state changes.

---

## Log Normalization

Raw logs from different sources use different formats — Windows XML, Linux text, Cisco syslog. Normalization maps them to a common schema so correlation rules work across sources.

Example: a failed logon from Windows (Event ID 4625) and a failed SSH login from Linux both get normalized to:

```
event_type = "authentication_failure"
src_ip     = <source IP>
user       = <attempted username>
timestamp  = <ISO 8601>
host       = <device hostname>
```

This common schema means a single correlation rule can detect brute-force attempts regardless of whether the target is Windows or Linux.

---

## Correlation Rules

### Brute force detection

```
If: more than 5 authentication_failure events
    for the same user
    from the same src_ip
    within 60 seconds

→ Alert: "Brute force attempt detected"
   Severity: HIGH
   Include: src_ip, user, count, first/last seen
```

### Privilege escalation

```
If: authentication_success for user X
    followed within 300 seconds by
    privilege_escalation (sudo/runas/UAC) for user X

→ Alert: "Privilege escalation after login"
   Severity: MEDIUM
```

### Lateral movement

```
If: same src_ip appears in authentication_success
    on more than 3 different hosts
    within 10 minutes

→ Alert: "Possible lateral movement"
   Severity: HIGH
```

### After-hours activity

```
If: authentication_success
    between 22:00 and 06:00 (local time)
    for a user with no after-hours history

→ Alert: "Off-hours login"
   Severity: LOW (informational)
```

---

## Dashboards

### Security overview

- Authentication events timeline (successes vs failures)
- Top failed login sources (by IP, by user)
- Active alerts by severity
- Geographic map of login sources (if GeoIP data available)

### System health (Zabbix)

Zabbix runs alongside the SIEM for infrastructure monitoring:

- CPU, memory, disk per host
- Process availability (SIEM services, forwarders)
- Network throughput
- Alert when a log source goes silent (forwarder down)

> 📷 *[Placeholder — SIEM dashboard screenshot]*

> 📷 *[Placeholder — Zabbix monitoring view]*

---

## Alert Tuning

The hardest part of SIEM work isn't writing rules — it's reducing false positives. An alert that fires 200 times a day for legitimate activity gets ignored.

Tuning steps for each rule:
1. Run the rule in audit mode — count how many times it fires
2. Identify the false positive patterns (scheduled tasks that look like new processes, service accounts with legitimate after-hours activity)
3. Add exclusions with justification: `NOT user IN ["svc_backup", "svc_monitoring"]`
4. Re-run, adjust threshold until the signal-to-noise ratio is acceptable
5. Document the exclusions — why each one exists, when to review it

---

## What I Learned

**Technical:**
- Log normalization is where most of the work is — raw logs are messy and inconsistent across sources
- Correlation rules that look simple on paper require careful tuning against real traffic to avoid alert fatigue
- Infrastructure monitoring (Zabbix) and security monitoring (SIEM) solve different problems but need to be integrated — knowing a forwarder is down is a security-relevant event
- Retention policies and storage costs matter at scale — deciding what to index vs. what to archive vs. what to discard is a real tradeoff

**Operational:**
- Alert fatigue is the biggest failure mode for a SIEM — too many low-quality alerts and the team stops looking
- Correlation rules need documentation: what they detect, what they don't detect, why exclusions exist
- A SIEM is only as good as its log sources — gaps in collection mean gaps in visibility

---

## Resources

- [Splunk documentation](https://docs.splunk.com/)
- [Zabbix documentation](https://www.zabbix.com/documentation/)
- [MITRE ATT&CK — detection coverage mapping](https://attack.mitre.org/)
- [Sigma — generic detection rule format](https://github.com/SigmaHQ/sigma)
- [Windows Security Event Log reference (SANS)](https://www.sans.org/posters/windows-forensic-analysis/)