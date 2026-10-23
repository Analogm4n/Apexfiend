# Apexfiend — Active Directory Penetration Testing Lab

Apexfiend is a self-built, fully isolated Active Directory penetration testing lab designed to simulate a realistic enterprise compromise — from an unauthenticated external foothold to full Domain Administrator-equivalent control of the environment (`corp.apexfiend.lab`).

The lab combines a vulnerable public-facing web application, realistic internal credential-hygiene weaknesses, and three independent, fully-chained Active Directory escalation paths (database/application, certificate services, and CI/CD automation) that converge on full domain compromise. It is built, tested, and documented end-to-end against real lab output — no step in this repository is speculative.

> ⚠️ **Lab disclaimer:** Apexfiend is a private, isolated lab environment built for offensive security training and portfolio purposes. It is not a real client engagement. Any report-style language (client name, confidentiality statements, etc.) found in `professional-technical-report/` is written in a professional format intentionally, to reflect real-world deliverable standards — it does not describe an actual organization or a real assessment.

## 📂 Repository Map

| Folder | Contents | Status |
|---|---|---|
| `docs/00-07/` | Lab architecture, Active Directory structure, network layout, and supporting infrastructure documentation | ✅ Complete |
| `docs/08-attack-paths/` | Reasoning-level narrative for each validated escalation path (initial access, web-to-VPN pivot, helpdesk escalation, SQL path, PKI path, Jenkins/RBCD path, full attack chain) | ✅ Complete |
| `professional-technical-report/` | Professional technical penetration test report — executive summary, risk ratings, findings, and remediation roadmap | ⏳ In progress |
| `writeups/` | Step-by-step technical walkthroughs with full exploitation detail, compiled from raw testing transcripts | ⏳ In progress |

## Where to Start

- Start with [`professional-technical-report`](./professional-technical-report/) — the professional report gives the full picture (scope, methodology, findings, and recommendations) without requiring you to read the raw exploitation steps.
- `docs/08-attack-paths/` documents how each escalation path was reasoned through and chained, and `writeups/` (once complete) will walk through the exact commands and tooling used.
- `docs/00-07/` covers the Active Directory forest, network segmentation, and host inventory the lab is built on.

## Attack Path Overview

The compromise begins at the external perimeter, pivots into the internal network via a developer workstation and VPN access, and then branches into **three independent, fully-validated privilege escalation paths** — any one of which alone is sufficient to reach full domain compromise:

1. **Database / Application path** — SQL injection in an internal web application's reporting functionality, chained through a linked server, to OS-level command execution.
2. **Certificate Services path** — A misconfigured certificate template combined with an NTLM-relayable enrollment endpoint (ESC8), used to obtain a certificate impersonating a privileged user.
3. **CI/CD & Delegation path** — Hijack of an automation service account via a legacy credential pattern, chained with over-permissioned Active Directory delegation, to achieve Resource-Based Constrained Delegation (RBCD) against a critical server.

Full detail on each path lives in `docs/08-attack-paths/` and the companion write-ups.

## Tooling

A full list of tools and techniques used across the engagement is maintained in `professional-technical-report/` (Appendix A).

## License / Usage

This repository documents a private lab environment built for educational and professional portfolio purposes. It does not target or reference any real organization, individual, or production system.

## Author

Jean Pierre Miranda Torres — Junior Penetration Tester.
