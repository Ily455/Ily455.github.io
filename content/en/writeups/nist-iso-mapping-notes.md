---
title: "NIST CSF v2 ↔ ISO 27002:2022 — Mapping Notes"
date: 2026-03-06
draft: false
description: "Notes from a mapping project between NIST Cybersecurity Framework v2 and ISO 27002:2022 controls, done during a GRC internship at Enedis. Includes the methodology, key alignment points, and a maturity simulation tool built on top of the mapping."
tags: ["grc", "nist", "iso27001", "compliance", "governance"]
---

> Work done during my final-year internship at **Enedis** (Aix-en-Provence, April–September 2025). The mission: produce a mapping between NIST CSF v2 and ISO 27002:2022, then build a maturity simulator on top of it to help teams assess their cybersecurity posture using either framework.

---

## Why This Mapping Matters

Organizations that operate internationally often face both frameworks simultaneously:

- **ISO 27001/27002** — dominant in Europe, required for certification, structured around an ISMS
- **NIST CSF** — widely used in the US, increasingly adopted globally, more operationally oriented

A company certified under ISO 27001 might also need to report against NIST CSF for US partners, or vice versa. Doing that manually — reading both frameworks and trying to figure out what maps to what — is tedious, error-prone, and inconsistent between teams.

A mapping lets you answer: *"If we score X on this ISO control, what does that imply about our NIST subcategory posture?"*

---

## The Frameworks

### NIST CSF v2

The Cybersecurity Framework was updated to v2 in 2024. It's organized around **6 Functions**:

```
GOVERN (GV)   ← new in v2 — organizational context, risk management strategy
IDENTIFY (ID) ← asset management, risk assessment
PROTECT (PR)  ← access control, data security, training
DETECT (DE)   ← anomalies, monitoring, detection processes
RESPOND (RS)  ← incident response, communications, analysis
RECOVER (RC)  ← recovery planning, improvements
```

Each function contains **Categories** and **Subcategories** — 106 subcategories in total in v2.

### ISO 27002:2022

ISO 27002 is the code of practice that supports ISO 27001. The 2022 version reorganized controls into **4 themes**:

```
Organizational controls  (37 controls)
People controls          (8 controls)
Physical controls        (14 controls)
Technological controls   (34 controls)
```

93 controls total. Each control has attributes (control type, cybersecurity concepts, operational capabilities, etc.) that help with categorization and mapping.

---

## Mapping Methodology

### Step 1 — Anchor on shared cybersecurity concepts

Both frameworks use similar underlying concepts. ISO 27002:2022 explicitly tags each control with cybersecurity concepts from the NIST CSF:

```
ISO attribute "Cybersecurity concepts":
  Identify / Protect / Detect / Respond / Recover
```

This gives a first-pass structural mapping — but it's coarse. One ISO control can map to multiple NIST subcategories, and one NIST subcategory may be partially addressed by several ISO controls.

### Step 2 — Semantic comparison

For each NIST subcategory, read the description and outcomes, then find the ISO controls that address the same requirement. This is manual work — the frameworks use different vocabulary for similar things.

Example:

```
NIST CSF v2 — PR.AA-01:
"Identities and credentials for authorized users, services, and hardware 
are managed by the organization."

Maps to ISO 27002:
  5.16 — Identity management
  5.17 — Authentication information
  8.2  — Privileged access rights
  8.5  — Secure authentication
```

### Step 3 — Relationship typing

Not all mappings are equivalent. Each relationship was typed:

| Type | Meaning |
|------|---------|
| **Full** | The ISO control fully addresses the NIST subcategory requirement |
| **Partial** | The ISO control addresses part of the requirement |
| **Complementary** | Multiple ISO controls together address the subcategory |
| **Directional** | ISO → NIST or NIST → ISO, not symmetric |

This distinction matters when using the mapping for maturity assessment — a partial match shouldn't be scored the same as a full match.

---

## Key Alignment Points

### Strong alignment areas

**Access control / Identity management**
NIST PR.AA (Protect: Identity Management) maps cleanly to ISO controls 5.15–5.18, 8.2, 8.5. Both frameworks treat access control as foundational — terminology differs but the requirements are nearly identical.

**Asset management**
NIST ID.AM (Identify: Asset Management) ↔ ISO 5.9, 5.10, 8.8. Both require inventory of assets and assignment of ownership. The NIST subcategories are slightly more granular on software and data assets.

**Incident response**
NIST RS.* ↔ ISO 5.24–5.28. The NIST Respond function maps well to the ISO incident management controls cluster. NIST is more prescriptive about communication and analysis phases.

### Gap areas

**GOVERN function (new in NIST v2)**
NIST v2 adds an entire new function around governance — organizational context, roles, policies, supply chain risk. ISO 27002 has equivalent controls (5.1–5.4, 5.19–5.22) but they're scattered across themes rather than grouped as a governance layer. The mapping here requires pulling from multiple ISO control clusters.

**Risk management**
NIST ID.RA (Risk Assessment) is more operationally detailed than ISO's risk treatment controls. ISO 27001 (the management standard, not 27002) handles the risk management process, which creates a gap when mapping against 27002 alone.

**Detection granularity**
The NIST DE function (Detect) is more granular than ISO's detection controls. ISO 8.15–8.16 (logging, monitoring) map to NIST DE broadly, but NIST distinguishes between event detection, continuous monitoring, and detection process improvement in ways ISO doesn't.

---

## The Maturity Simulator

On top of the mapping, a tool was built to let teams estimate their NIST CSF maturity from ISO 27002 assessment results (or vice versa).

### How it works

```
Input: ISO control scores (0–4 scale, based on ISO 27001 audit grades)
      ↓
Mapping table (control → subcategory relationships with weights)
      ↓
Weighted aggregation per NIST subcategory
      ↓
Output: Estimated NIST CSF maturity tier per function/category
```

The scoring scale aligns with NIST CSF Tiers (1–4) and ISO maturity levels (Not implemented → Partially → Largely → Fully implemented).

### Weighting logic

A full ISO→NIST relationship contributes 100% of the ISO score to the subcategory. A partial relationship contributes 50–75% depending on the semantic overlap assessed during the mapping phase.

```python
# Simplified
nist_score = sum(
    iso_score[control] * weight
    for control, weight in mapping[subcategory]
) / total_weight
```

### Output example

```
Function: PROTECT
  PR.AA (Identity Mgmt)     — Estimated Tier: 3 (Repeatable)
  PR.AT (Awareness)         — Estimated Tier: 2 (Informing)
  PR.DS (Data Security)     — Estimated Tier: 3 (Repeatable)
  PR.PS (Platform Security) — Estimated Tier: 2 (Informing)
  PR.IR (Resilience)        — Estimated Tier: 1 (Partial)
```

---

## Limitations

**It's an estimate, not an audit.** The mapping can tell you approximately where you likely stand on NIST given your ISO posture — it can't replace an actual NIST CSF assessment done by someone who knows your organization.

**Mapping is subjective.** Semantic comparison involves judgment. Two people mapping the same pair of controls may not agree on whether the relationship is "full" or "partial." This introduces variability.

**The frameworks aren't equivalent.** They were designed for different contexts with different assumptions. Some NIST subcategories have no good ISO equivalent, and some ISO controls address things NIST doesn't.

**ISO 27002 ≠ ISO 27001.** ISO 27001 is the management system standard. ISO 27002 is just the control catalog. If you only have 27002 assessment data, you're missing the process and ISMS-level controls that ISO 27001 includes.

---

## What I Learned

**On frameworks:**
- NIST CSF v2's GOVERN function is a meaningful addition — it forces organizations to think about cybersecurity at the strategic level, not just operationally
- ISO 27002:2022 is a significant improvement over the 2013 version — the attribute tagging makes mappings much easier to bootstrap
- No mapping is perfect; the value is in making cross-framework communication faster and more consistent, not eliminating interpretation

**On GRC work in practice:**
- Most of the effort is in the semantic comparison — technical understanding matters even in governance roles
- Tools built on top of frameworks are only as good as their underlying mapping — garbage in, garbage out
- Communication is the job: the technical output has to be explainable to non-technical stakeholders

---

## Resources

- [NIST CSF v2 — official documentation](https://www.nist.gov/cyberframework)
- [NIST CSF v2 — reference tool](https://csrc.nist.gov/projects/cybersecurity-framework)
- [ISO/IEC 27002:2022](https://www.iso.org/standard/75652.html)
- [NIST → ISO 27001 crosswalk (NIST SP 800-53)](https://csrc.nist.gov/projects/risk-management/sp800-53-controls/mappings)
- [EBIOS RM methodology (ANSSI)](https://www.ssi.gouv.fr/guide/ebios-risk-manager-the-method/)