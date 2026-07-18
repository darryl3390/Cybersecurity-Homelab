# Cybersecurity & IT Home Lab

**Built by:** Darryl Briggs
**Platform:** iMac 2017 · Intel Core i7 (4C/8T) · 48GB RAM · VirtualBox
**Goal:** Build a fully functional practice environment that replicates real enterprise IT and security infrastructure — hands-on skill-building in support of a career transition into IT and cybersecurity.

---

## Why This Lab Exists

I'm transitioning into IT and cybersecurity from a background in education and school building administration. That work involved managing systems, people, and risk at an operational level — but not the hands-on technical skills this field requires. Certifications prove knowledge — this lab proves application. Every VM, configuration, and exercise here is something I built, broke, troubleshot, and documented myself.

---

## Lab Evolution: Phase 1 → Phase 2

This lab didn't launch as its current architecture — it got here by identifying its own limitations and rebuilding around them, and both phases are kept in this same repo on purpose.

- **Phase 1** established the foundation: Active Directory domain, a handful of VMs on a single flat network, and the first vulnerability management cycle (Nessus scan → remediate → verify).
- **Phase 2** is a deliberate re-architecture, prompted by a real gap: a flat network gave zero boundary enforcement, no firewall rules to write, and no perimeter logging. Phase 2 introduces network segmentation behind a firewall/router, a SIEM, and a dedicated GRC platform — closer to how an actual small enterprise environment is structured.

The full reasoning behind each decision — not just what was built, but why — is in the [build log](#build-log) below. The progression from Phase 1 to Phase 2 is intentionally visible rather than cleaned up, since identifying your own architecture's limitations and fixing them is itself part of the skill this lab is meant to demonstrate.

---

## Certifications

| Cert | Status |
|---|---|
| CompTIA A+ | ✅ Complete |
| CompTIA Network+ | ✅ Complete |
| CompTIA Security+ | ✅ Complete |

---

## Current Lab Architecture (Phase 2)

The network is segmented into three isolated zones, with **pfSense** as the only device routing between them — nothing crosses a boundary without an explicit firewall rule.

| Segment | Purpose | VM(s) | Tier |
|---|---|---|---|
| **ATTACK** | Adversary simulation | Kali Linux | Rotating |
| **AD_LAB** | Enterprise identity + endpoints | DC01 (Windows Server — AD DS, DNS), WIN11 (domain workstation), Ubuntu Desktop (Linux target) | Rotating |
| **DEFENDER** | Logging, monitoring, governance, and controlled egress | Ubuntu Server (Splunk + Universal Forwarders), Eramba (GRC platform), APT-Proxy (apt-cacher-ng) | Splunk: always-on · Eramba: rotating (GRC sessions only) · APT-Proxy: on-demand (AD_LAB patch windows) |

**pfSense** sits with an interface on all three segments (plus WAN/NAT), enforcing rules like *ATTACK → AD_LAB allow*, *AD_LAB → DEFENDER allow (log forwarding)*, *AD_LAB → WAN block*, default deny-all.

**Domain:** `homelab.local`

A full network diagram with design-decision rationale (including deliberate trade-offs like VirtualBox Internal Networks standing in for true VLANs) is included in this repo — see `/diagrams/homelab-network-diagram.html`.

> **Phase 1 architecture (historical):** a single flat VirtualBox host-only network with no segmentation or firewall. Preserved in the early build log entries for reference — see the [build log](#build-log).

---

## Tools In Use

| Tool | Purpose | Status |
|---|---|---|
| **Active Directory Domain Services** | Identity and access management | ✅ Deployed (Phase 1) |
| **pfSense** | Firewall / router enforcing network segmentation | ✅ Deployed (Phase 2) |
| **Splunk Enterprise** | SIEM — log aggregation, correlation, detection | ✅ Deployed, integrated into segmented architecture (Phase 2) |
| **Nessus Essentials** | Vulnerability scanning | ✅ In active use — full scan/remediate/verify cycles completed |
| **Eramba** | Open-source GRC platform — risk register, control mapping, policy documents | ✅ Deployed (Phase 2) — populated with real credentialed-scan findings |
| **Kali Linux** | Attack simulation / security tooling | ✅ Deployed |
| **APT-Proxy (apt-cacher-ng)** | Dedicated internal package proxy — controlled egress for AD_LAB, avoiding a standing internet route | ✅ Deployed (Phase 2) |

---

## Frameworks Referenced

This lab is mapped to established frameworks rather than built ad hoc:

- **NIST Cybersecurity Framework (CSF) 2.0** — Identify, Protect, Detect, Respond, Govern; Recover is a known, tracked gap (backup/restore drill scheduled, not yet completed)
- **CIS Critical Security Controls v8** — secure configuration, vulnerability management, network infrastructure and monitoring, audit log management
- **NIST SP 800-61** — incident response lifecycle informing the detection → investigation → write-up workflow
- **MITRE ATT&CK** — detections tagged by technique ID

---

## Build Log

The full, dated build log — including what was built, what broke, root cause analysis, and what was learned — lives in `/build-log.md`. Entries are numbered per phase (Phase 1: Entries 001+, Phase 2: Entries 001+), with Phase 2 explicitly starting its own sequence to reflect the architectural restart, while remaining in this same repo for continuity.

---

## Repo Structure

```
/
├── README.md                    # this file
├── build-log.md                 # full dated build log, Phase 1 and Phase 2
├── exercise-logs/                # write-ups of individual lab exercises and sessions
├── grc-artifacts/                 # risk register summaries, framework mappings, policies
│   └── (Phase 2 additions: control mappings, SOC 2 crosswalk, remediation tracker)
├── screenshots/                  # supporting screenshots referenced in the build log
└── diagrams/                     # (Phase 2 addition) current network architecture
    └── homelab-network-diagram.html
```
