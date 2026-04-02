---
title: "ICS/OT Threat Modeling — Three Critical Infrastructure Attacks"
date: 2025-03-26
draft: false
description: "Structured threat analysis of three ICS/OT attacks using MITRE ATT&CK for ICS: Stuxnet (2010), Ukraine power grid (2015), and Danish energy sector (2023). Attack chain reconstruction, TTPs mapping, and framework comparison."
tags: ["ics", "ot", "scada", "mitre-attack", "threat-modeling", "stuxnet", "critical-infrastructure"]
---

> Group academic project — AMU M2 FSI, 2023–2024. Focus: IT/OT convergence security in the energy sector. This writeup covers the threat modeling methodology applied to three landmark ICS incidents using MITRE ATT&CK for ICS.

---

## Context: IT/OT Convergence

Operational Technology (OT) — SCADA systems, PLCs, industrial control systems — was historically air-gapped. IT/OT convergence (IoT sensors, cloud connectivity, remote access) has changed that: the same network paths that improve operational efficiency create attack surface.

The key asymmetry: OT systems prioritize availability over confidentiality. Patching is constrained by uptime requirements. Legacy systems run for 20+ years. A vulnerability that would be a minor incident in IT can cause physical damage and service disruption in OT.

---

## Methodology

Each incident was mapped against **MITRE ATT&CK for ICS** (distinct from the enterprise framework). Key tactics relevant to ICS attacks:

| Tactic ID | Tactic Name | Description |
|-----------|-------------|-------------|
| TA0108 | Initial Access | Entry into the IT or OT network |
| TA0104 | Execution | Running adversary-controlled code on targets |
| TA0110 | Persistence | Maintaining foothold across reboots/updates |
| TA0111 | Privilege Escalation | Gaining elevated access |
| TA0103 | Lateral Movement | Moving from IT toward OT networks |
| TA0102 | Collection | Gathering process data, configurations |
| TA0109 | Inhibit Response Function | Disabling safety systems, alarms |
| TA0105 | Impact | Physical disruption, destruction |

---

## Case 1 — Stuxnet (2010)

### Background

Stuxnet is the first known cyberweapon designed to cause physical destruction. Discovered in June 2010, it targeted Iranian nuclear centrifuges at the Natanz enrichment facility. The worm exploited **4 zero-day vulnerabilities** simultaneously and contained approximately 150,000 lines of code — an order of magnitude more complex than any malware seen at the time.

### Attack Chain

**Initial Access — Removable Media (T0819)**
Stuxnet spread via infected USB drives. The target (Natanz) was air-gapped from the internet; physical media was the only entry vector. The worm exploited a Windows Shell LNK vulnerability (CVE-2010-2568) that triggered on simply viewing the drive contents — no user execution required.

**Execution — Exploiting four zero-days**
- CVE-2010-2568: Windows Shell LNK (zero-day)
- CVE-2010-2772: Windows Task Scheduler (zero-day)
- CVE-2010-2729: Windows Print Spooler (zero-day)
- CVE-2010-2772: Windows Server Service (zero-day)

**Persistence + Rootkit**
Stuxnet used a rootkit (signed with stolen Realtek and JMicron certificates) to hide its files and registry entries. It modified the PLC ladder logic while masking its presence from the Siemens WinCC SCADA interface.

**Lateral Movement — targeting SIMATIC S7 PLCs**
The payload specifically targeted Siemens S7-315 and S7-417 PLCs connected via Siemens Step 7 software. It intercepted communications between the engineering workstation and the PLCs.

**Impact — Physical destruction (T0879)**
The centrifuge control logic was modified to spin rotors at abnormal frequencies (1,410 Hz then 2 Hz, then 1,064 Hz) while reporting normal operation to operators. Approximately 1,000 centrifuges were damaged or destroyed. Operators saw no alarms.

| ATT&CK Tactic | Technique | Detail |
|---------------|-----------|--------|
| TA0108 Initial Access | T0819 Removable Media | USB LNK exploit, air-gap bypass |
| TA0104 Execution | T0807 Command-Line Interface | Four zero-days chained |
| TA0110 Persistence | T0873 Project File Infection | Ladder logic modification + rootkit |
| TA0103 Lateral Movement | T0843 Program Download | Step 7 interception |
| TA0109 Inhibit Response | T0838 Modify Alarm Settings | Masking abnormal rotor readings |
| TA0105 Impact | T0879 Damage to Property | ~1,000 centrifuges destroyed |

### Key takeaways

The air gap was not a security boundary — it was bypassed via the supply chain (infected USB drives circulating among contractors). The rootkit demonstrating that a sufficiently sophisticated adversary can maintain plausible deniability while causing physical damage. Four simultaneous zero-days signal a nation-state actor with significant resources.

---

## Case 2 — Ukraine Power Grid Attack (December 2015)

### Background

On December 23, 2015, coordinated cyberattacks cut power to approximately **225,000 customers** across three Ukrainian regional distribution companies (Kyivoblenergo, Prykarpattyaoblenergo, Chernivtsioblenergo). Attributed to Sandworm (APT44, GRU Unit 74455). The first confirmed cyberattack to cause a power outage.

### Attack Chain

**Initial Access — Spear Phishing (T0865)**
Employees received targeted emails containing malicious Word documents with BlackEnergy malware. BlackEnergy 3 was delivered via macros.

**Execution + Persistence**
BlackEnergy provided a persistent backdoor. Attackers maintained access for **months** before the actual disruption — conducting reconnaissance, harvesting VPN credentials, and mapping the SCADA environment.

**Lateral Movement — VPN pivot**
Using harvested VPN credentials, attackers moved from the IT network into the OT network. The VPN lacked multi-factor authentication.

**Collection**
Attackers gathered SCADA operator screenshots, substation configurations, and learned normal operating procedures. They waited.

**Impact — Coordinated grid disruption (T0879)**
On December 23:
1. Remote access to 3 distribution companies' SCADA/HMI systems simultaneously
2. Operators' control interfaces seized via UltraVNC remote desktop
3. 30 substations taken offline manually through the SCADA interface
4. **KillDisk** deployed to overwrite MBR on SCADA workstations and RTUs — rendering systems unbootable after the attack
5. Phone lines flooded with calls to prevent operators from reporting the outage

The combination of SCADA manipulation + KillDisk + phone flooding shows a deliberate layering of impact and response suppression.

| ATT&CK Tactic | Technique | Detail |
|---------------|-----------|--------|
| TA0108 Initial Access | T0865 Spearphishing Attachment | BlackEnergy 3 via Word macro |
| TA0110 Persistence | T0891 Hardcoded Credentials | VPN credentials harvested |
| TA0103 Lateral Movement | T0859 Valid Accounts | VPN pivot to OT network |
| TA0102 Collection | T0801 Monitor Process State | Operator screen harvesting |
| TA0109 Inhibit Response | T0804 Block Reporting Message | Phone line flooding |
| TA0105 Impact | T0879 Damage to Property | KillDisk, 30 substations offline |

### Key takeaways

The absence of MFA on VPN was the critical pivot point. Months of dwell time allowed the attackers to understand the environment thoroughly before acting. KillDisk was deployed not to cause the outage but to slow recovery — the outage itself was achieved through normal SCADA operations. The attack required both technical capability and operational planning.

---

## Case 3 — Danish Energy Sector (May 2023)

### Background

In May 2023, **22 Danish energy companies** were attacked in a coordinated two-wave campaign. Documented by SektorCERT (Danish critical infrastructure CERT). The attack exploited three Zyxel firewall vulnerabilities patched between April and May 2023 — attackers moved before many operators had applied patches.

### Vulnerabilities Exploited

| CVE | CVSS | Type | Affected Products |
|-----|------|------|-------------------|
| CVE-2023-28771 | 9.8 | OS Command Injection (unauthenticated) | Zyxel ATP, USG FLEX, VPN, ZyWALL |
| CVE-2023-33009 | 9.8 | Buffer Overflow (pre-auth) | Same product lines |
| CVE-2023-33010 | 9.8 | Buffer Overflow (pre-auth) | Same product lines |

All three are unauthenticated pre-auth vulnerabilities — no credentials needed to exploit.

### Two-Wave Attack

**Wave 1**: Opportunistic exploitation of CVE-2023-28771 across 11 companies. Command injection via crafted IKEv2 packets. Attackers gained shell access to firewall devices at the network perimeter. Some companies' industrial control systems were directly reachable from the compromised firewalls.

**Wave 2** (days later): Targeted follow-up against companies with confirmed OT network access. The second wave used Mirai botnet infrastructure and attempted DDoS alongside persistence mechanisms on the firewalls. Several companies were forced to disconnect from the internet entirely.

SektorCERT's rapid response (coordinating across all 22 companies simultaneously, sharing IoCs in real-time) limited the blast radius. No confirmed physical impact on grid operations, but several companies lost remote monitoring capability during the incident.

| ATT&CK Tactic | Technique | Detail |
|---------------|-----------|--------|
| TA0108 Initial Access | T0866 Exploitation of Remote Services | CVE-2023-28771 unauthenticated RCE |
| TA0104 Execution | T0807 Command-Line Interface | OS command injection via IKEv2 |
| TA0110 Persistence | T0891 Hardcoded Credentials | Firewall persistence mechanisms |
| TA0103 Lateral Movement | T0843 Program Download | Pivot toward OT networks |
| TA0109 Inhibit Response | T0804 Block Reporting Message | DDoS + internet disconnection forced |
| TA0105 Impact | T0879 Damage to Property | Remote monitoring loss, potential OT access |

### Key takeaways

The patch gap — 22 companies running vulnerable Zyxel firmware after patches were available — shows the operational challenge of patch management in critical infrastructure. The two-wave structure (opportunistic scan → targeted follow-up) is consistent with a reconnaissance-then-exploitation model. SektorCERT's coordinated response is a reference case for sector-level incident response.

---

## Cross-Case Analysis

| Dimension | Stuxnet | Ukraine 2015 | Denmark 2023 |
|-----------|---------|--------------|--------------|
| Actor | Nation-state | Nation-state (Sandworm/GRU) | Unknown (Sandworm suspected) |
| Initial vector | Physical (USB) | Spear phishing | Unpatched CVEs |
| Dwell time | Months | Months | Days |
| OT access method | Supply chain | VPN pivot | Firewall exploit |
| Physical impact | Yes (~1,000 centrifuges) | Yes (225k customers) | No (near-miss) |
| Defender response | Delayed (covert op) | Manual restoration | Coordinated CERT |

**Common patterns across all three:**
- IT network compromise precedes OT impact
- Dwell time used for reconnaissance before action
- Safety/monitoring systems explicitly targeted or bypassed
- Recovery was complicated by deliberate anti-forensic or persistence mechanisms

---

## What This Analysis Shows

Mapping these attacks to MITRE ATT&CK for ICS reveals that despite the years between them, the attack patterns are structurally similar: the IT/OT boundary is crossed via a different method each time (USB, VPN, firewall), but the subsequent reconnaissance → impact sequence is consistent.

The framework also highlights where defenses would have the highest leverage: Initial Access (MFA, patch management, removable media controls) and Lateral Movement (network segmentation, OT/IT boundary enforcement) appear in all three cases as the critical chokepoints.

---

## Resources

- [MITRE ATT&CK for ICS](https://attack.mitre.org/matrices/ics/)
- [SektorCERT — Danish Energy Sector Report (2023)](https://sektorcert.dk/wp-content/uploads/2023/11/SektorCERT-The-attack-against-Danish-critical-infrastructure-TLP-WHITE.pdf)
- [ICS-CERT — Ukraine Power Grid Attack Analysis](https://www.cisa.gov/uscert/ics/alerts/IR-ALERT-H-16-056-01)
- [Langner — To Kill a Centrifuge (Stuxnet technical analysis)](https://www.langner.com/to-kill-a-centrifuge/)
- [IEC 62443 — Industrial Automation and Control Systems Security](https://www.iec.ch/iecnow/iecnow-smart-manufacturing/iec-62443/)
