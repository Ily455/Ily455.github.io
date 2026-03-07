---
title: "Secure Network Architecture on ESXi"
date: 2026-03-06
draft: false
description: "Design and deployment of a segmented enterprise network on VMware ESXi — VLANs, pfSense firewall with strict ACLs, DMZ, VPN, and a monitored security perimeter — entirely virtualized."
tags: ["network", "esxi", "vmware", "pfsense", "vlan", "firewall", "infrastructure"]
---

## Overview

This project was the design and deployment of a complete enterprise-grade network architecture, running entirely on VMware ESXi. The focus was on security: network segmentation with VLANs, a pfSense firewall enforcing strict access control, a proper DMZ for exposed services, and a VPN gateway for remote access.

Everything that would normally require physical switches, routers, and firewall appliances was virtualized — which also made it possible to test attack scenarios and firewall rule changes without risking physical infrastructure.

---

## Architecture

```
Internet (WAN)
      │
      ▼
┌─────────────┐
│   pfSense   │  ← Firewall, router, VPN gateway, IDS/IPS
└──────┬──────┘
       │
   ┌───┴────────────────────────────────────┐
   │              vSwitch (ESXi)            │
   └───┬──────────┬──────────┬──────────┬──┘
       │          │          │          │
  VLAN 10    VLAN 20    VLAN 30    VLAN 99
  LAN/Corp   DMZ        Mgmt       Isolated
       │          │          │          │
  ┌────┴───┐ ┌────┴───┐ ┌────┴───┐ ┌────┴───┐
  │Workst. │ │Web srv │ │ESXi   │ │Sandbox │
  │AD DC   │ │Mail srv│ │vCenter│ │VMs     │
  │File srv│ │        │ │       │ │        │
  └────────┘ └────────┘ └────────┘ └────────┘
```

---

## Network Segments

### VLAN 10 — Corporate LAN

Internal workstations and servers. This segment has:
- Access to internal resources (file server, AD)
- Outbound internet access through pfSense (HTTP/HTTPS only by default)
- No direct access to DMZ or management segments
- Active Directory domain controller for authentication

### VLAN 20 — DMZ

Services that need to be reachable from the internet. Strict rules:
- Internet → DMZ: only ports 80 and 443 (web), 25/465/587/993 (mail)
- DMZ → LAN: **blocked** — a compromised DMZ host cannot reach internal resources
- DMZ → DMZ: controlled — services can communicate on specific ports only

### VLAN 30 — Management

ESXi host, vCenter, pfSense management interface. Most restricted segment:
- Only accessible from a dedicated admin workstation on VLAN 10
- No inbound connections from DMZ or internet
- Separate credentials from the corporate domain

### VLAN 99 — Isolated / Sandbox

No routing to any other segment. Used for:
- Testing new VMs before deployment
- Running potentially malicious samples (isolated from everything)
- Lab experimentation

---

## pfSense Configuration

### Firewall rules — key policies

The default pfSense rule set was replaced with explicit allow rules and a default-deny policy.

**WAN interface:**
```
allow  TCP  any → DMZ:80,443     ← public web services
allow  TCP  any → DMZ:25,465,587,993  ← mail
allow  UDP  any → WAN:1194       ← VPN
block  any  any → any            ← everything else
```

**LAN → WAN:**
```
allow  TCP  LAN → any:80,443     ← HTTP/HTTPS
allow  UDP  LAN → any:53         ← DNS
block  any  LAN → any            ← everything else (no arbitrary ports out)
```

**DMZ → LAN:**
```
block  any  DMZ → LAN            ← DMZ cannot reach internal network
```

**Mgmt → any:**
```
allow  TCP  admin_ws → pfSense:443    ← web UI
allow  TCP  admin_ws → ESXi:443       ← ESXi management
block  any  → Mgmt:any               ← no inbound to management segment
```

### NAT

Outbound NAT masquerades LAN and DMZ traffic behind the WAN IP. Port forwarding on WAN routes incoming traffic to the appropriate DMZ host.

### VPN — OpenVPN

Remote access via OpenVPN on UDP 1194. Configuration:
- Certificate-based authentication (no password-only auth)
- Remote clients land in a dedicated VPN subnet
- Split tunneling disabled — all traffic routed through the VPN (traffic that doesn't need to be secured shouldn't be routed through the enterprise network)
- VPN clients get access to VLAN 10 resources only, same rules as local LAN

### IDS/IPS — Snort on pfSense

Snort runs inline on the WAN interface to inspect incoming traffic:
- Rulesets: ET Open, Snort community rules
- Alerts on known exploit patterns, port scans, malformed packets
- Blocking mode for high-confidence rules

> 📷 *[Placeholder — pfSense dashboard / firewall rules view]*

---

## ESXi Virtual Networking

VMware ESXi handles the virtual switch and VLAN tagging. Each VM is connected to a port group that corresponds to a VLAN.

```
ESXi Physical NIC (uplink)
    │
    └── vSwitch0
         ├── Port Group: "LAN"     (VLAN 10)
         ├── Port Group: "DMZ"     (VLAN 20)
         ├── Port Group: "Mgmt"    (VLAN 30)
         └── Port Group: "Sandbox" (VLAN 99)
```

pfSense runs as a VM with virtual interfaces connected to each port group — it sees four network interfaces and acts as the inter-VLAN router and firewall for all traffic.

---

## Security Testing

With the environment fully virtualized, it was possible to run attack scenarios without risk:

**Port scanning from DMZ to LAN:**
```bash
nmap -sS 192.168.10.0/24   ← from a DMZ VM
# All hosts should appear as "filtered" — pfSense is blocking
```

**Attempted lateral movement:**
Simulating a compromised DMZ web server trying to reach AD DC — all connections blocked, Snort alerts fire.

**Firewall rule change testing:**
Add a rule, verify it allows the intended traffic, verify it doesn't open anything unintended, then document and commit.

---

## What I Learned

**Network:**
- VLAN segmentation is the foundation — if your network is flat, any compromise becomes a full compromise
- Default-deny firewall policy is the only sensible approach — default-allow with exceptions always has gaps
- DMZ design: the key invariant is that a compromised DMZ host cannot reach internal resources directly

**Virtualization:**
- ESXi virtual networking is a full software-defined network — vSwitch, port groups, and VLAN tagging work the same as physical gear
- Running the firewall as a VM on the same host it's protecting is a design limitation — in production, the firewall should be on separate hardware or a separate ESXi host

**Security operations:**
- IDS rules without tuning generate too much noise — the Snort setup required the same kind of alert tuning as the SIEM rules
- Every firewall rule should have a documented justification and a review date
- Access to the management segment is the highest-value target — protecting vCenter and ESXi is as important as protecting the data

---

## Resources

- [pfSense documentation](https://docs.netgate.com/pfsense/en/latest/)
- [VMware ESXi documentation](https://docs.vmware.com/en/VMware-vSphere/)
- [OpenVPN on pfSense](https://docs.netgate.com/pfsense/en/latest/vpn/openvpn/)
- [Snort rules documentation](https://www.snort.org/documents)
- [VirtualBox / VMware networking — concepts](https://www.vmware.com/topics/glossary/content/virtual-networking.html)