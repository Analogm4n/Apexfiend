# Apexfiend — AD CS Architecture

## 1. Purpose

This document provides a consolidated architectural overview of the lab's Active Directory Certificate Services deployment, synthesizing `ca02.md` and `certificate-templates.md`.

## 2. Topology

```text
Forest:           apexfiend.lab
CA host:           CA02.corp.apexfiend.lab   (172.16.50.103, VLAN 50)
CA logical name:    corp-CA01-CA
CA type:             Enterprise Root CA — single-tier, no subordinate CA
Enrollment object:   CN=corp-CA01-CA,CN=Enrollment Services,CN=Public Key Services,CN=Services,CN=Configuration,DC=apexfiend,DC=lab
```

A single-tier Enterprise Root CA is itself a simplification relative to many real enterprise PKI deployments (which typically separate an offline root CA from one or more issuing/subordinate CAs). This is a reasonable lab simplification and not treated as a finding.

## 3. Certificate Templates

Published templates fall into two categories (`ca02.md`, Section 5):

- **Built-in Microsoft defaults** (`User`, `Machine`, `WebServer`, `DomainController`, `KerberosAuthentication`, `EFS`, `EFSRecovery`, `SubCA`, `Administrator`, `DirectoryEmailReplication`, `DomainControllerAuthentication`) — present on any Enterprise CA, unmodified.
- **Custom/lab-added templates**: `UserTemporary` (the one template of actual interest — see below), plus `esc8`, `test`, `test-p`, `Mail-Server`, `GiteaWebServer`. Of these, `esc8`/`test`/`test-p` are confirmed intentional noise — decoy templates with plausible-sounding names, not themselves vulnerable or exploited — with more planned to be added by the operator over time. `Mail-Server` and `GiteaWebServer` are presumed production templates backing TLS certificates for `MAIL01`/`GITEA01` respectively.

**`UserTemporary` is the only template with a role in the documented attack path.** It is a near-copy of the default `User` template, differing only in two added ACEs (`certificate-templates.md`, Section 3): read-only access for `PKI-Operations` (`j.derek`'s group), and an individually-granted Enroll right for `m.ortega`. Combined with its Client Authentication EKU and lack of a manager-approval requirement, this enables the **ESC8** technique (NTLM relay to the CA's HTTP enrollment endpoint) — confirmed by the operator as the intended and correct technique, matching `00-overview/attack-path.md` Stage 7.

## 4. Web Enrollment (HTTP)

**Confirmed installed and functional.** The Web Enrollment role is present on `CA02`, and the ESC8 NTLM-relay technique has been tested end-to-end by the operator — including successful recovery of a certificate issued to `m.ortega` via the relayed request. This closes the open item from the previous revision of this document and confirms `00-overview/attack-path.md` Stage 7 as validated, not merely designed.

## 5. Certificate Inventory Tooling

A separate component — referenced in `00-overview/lab-setup.md` as a scheduled task running under `j.derek`'s context on `MSSQL02`, which queries certificate/template information and copies a report to `WK03` — has not yet been captured against live configuration. See `certificate-inventory.md` for what's confirmed once that data is available.

## 6. Security Summary

| Component | Status |
|---|---|
| CA tier structure | Single-tier Enterprise Root — simplification, not a finding |
| `UserTemporary` template | Confirmed ESC8-exploitable; `m.ortega` the specific target identity |
| `PKI-Operations`/`j.derek` access | Confirmed read-only, as designed |
| Web enrollment (HTTP) availability | Not yet independently verified |
| Decoy/noise templates | Present by design; more planned |

## 7. Validation Status

| Item | Status |
|---|---|
| CA topology and logical identity | Verified |
| Template categorization (default vs. custom vs. noise) | Verified (operator-confirmed) |
| ESC8 as the applicable technique | Confirmed by operator |
| Web enrollment role installed/enabled on CA02 | Not yet verified |
| Certificate inventory tooling configuration | Not yet captured |

## 8. Related Documentation

- `ca02.md` — CA-level configuration detail
- `certificate-templates.md` — `UserTemporary` flag and ACL analysis
- `certificate-inventory.md` *(pending)* — Inventory tooling detail
- `../00-overview/attack-path.md` — Stages 6–7
- `../../pki-documentation/` *(separate repository)* — Already corrected by the operator to reflect ESC8
