# Apexfiend — Network Segmentation

## 1. Purpose

This document records the **intended** segmentation model between the lab's VLANs — the rules the firewall is meant to enforce once fully configured. Unlike `02-active-directory/`, which records verified live configuration, this document currently records **design intent**, not independently tested results.

Firewall configuration between some VLANs is still in progress. Until each rule below has been tested and confirmed (see `04. Validation Status`), this document should be read as the target state, not a verified one. Entries should be updated to "Verified" individually as rules are completed and tested, rather than all at once.

## 2. DMZ Egress Policy

```text
Segment: VLAN 30 — DMZ (WEB01, WEB02)
```

| Direction | Rule |
|---|---|
| Outbound from DMZ | Permitted only on TCP 80 (HTTP) and TCP 443 (HTTPS) |
| Inbound to DMZ (from corporate segments) | Not permitted by default — DMZ hosts are not domain-joined and have no legitimate reason to accept connections initiated from inside the corporate network |
| DMZ to corporate segments (VLAN 20, 40, 50, 60) | Not permitted — no defined application dependency requires it (see `01-architecture/infrastructure-dependencies.md`) |

This is the DMZ's intended isolation posture: it can reach the outside world (or be reached from the outside world, subject to WAN-side rules not covered here) on web ports only, and has no route into the corporate environment at all. A compromise of WEB01 or WEB02 should not, by network access alone, provide any path toward WEB03, the domain controllers, SQL Server, or CA01.

## 3. VLAN-to-Domain-Controller Policy

```text
Segments: VLAN 20 (DC02, DC03), VLAN 10 (DC01)
```

The intended rule is that every domain-joined segment can reach the domain controllers **only** on the ports required for standard Active Directory client operations — not unrestricted access to VLAN 10/20.

| Service | Port(s) | Protocol |
|---|---|---|
| DNS | 53 | TCP/UDP |
| Kerberos | 88 | TCP/UDP |
| Kerberos password change | 464 | TCP/UDP |
| LDAP | 389 | TCP/UDP |
| LDAP (SSL) | 636 | TCP |
| Global Catalog LDAP | 3268 | TCP |
| Global Catalog LDAP (SSL) | 3269 | TCP |
| SMB / file replication | 445 | TCP |
| RPC Endpoint Mapper | 135 | TCP |
| RPC dynamic range | 49152–65535 | TCP |
| NTP | 123 | UDP |

This set applies to every domain-joined segment that needs to authenticate against the domain — `VLAN 40` (WEB03, Gitea, SQL infrastructure), `VLAN 50` (CA01), and `VLAN 60` (workstations). Segments and hosts that are **not** domain-joined (WEB01, WEB02 in the DMZ) have no rule permitting any of this traffic at all, per Section 2.

Access from VLAN 10/20 outward (domain controller to client) is not separately modeled here; DC-initiated connections to clients are primarily Group Policy and replication-related and are assumed permitted within the domain-joined segments.

## 4. Other Inter-VLAN Rules (Intended)

These follow directly from the dependencies already documented in `01-architecture/infrastructure-dependencies.md` and are restated here as the segmentation rules that should enforce them:

| From | To | Intended access |
|---|---|---|
| VLAN 60 (workstations) | VLAN 50 (CA01) | Certificate enrollment ports only (HTTPS 443, and the standard AD ports in Section 3) |
| VLAN 40 (WEB03/Gitea/SQL) | VLAN 50 (CA01) | Required only where a documented dependency exists (certificate inventory tooling) |
| VLAN 60 (workstations) | VLAN 40 (application/data) | Application-specific ports only (Gitea web/SSH, Mattermost/osTicket on WEB03) — no direct SQL access from ordinary workstations |
| VLAN 40 (WEB03) | VLAN 40 (MSSQL01, MSSQL02) | SQL Server (TCP 1433, or configured instance port) |
| `10.0.200.0/24` (VPN clients) | VLAN 60 (workstations) | Permitted, per the VPN's intended role as the remote-access entry point |
| `10.0.200.0/24` (VPN clients) | All other VLANs | Not permitted directly — VPN clients reach other segments only by first reaching a workstation or jumping through an application, not via a flat route to every VLAN |

## 5. Validation Status

| Rule | Status |
|---|---|
| DMZ egress restricted to 80/443 | Assumed (design intent) — not yet independently tested |
| DMZ has no route to corporate segments | Assumed — not yet independently tested; firewall rules between some VLANs are still being configured |
| Domain-joined segments restricted to standard AD ports against DC01/DC02/DC03 | Assumed — not yet independently tested |
| VPN clients restricted to VLAN 60 only | Assumed — not yet independently tested |
| Other inter-VLAN rules (Section 4) | Assumed — not yet independently tested |

None of the rules in this document should be treated as confirmed until tested with the source/destination/port methodology described in `01-architecture/firewall-and-routing.md`, Section 8. This document will be updated — rule by rule, not as a single pass — as the firewall configuration is completed and each rule is verified.

## 6. Related Documentation

- `pfsense.md` — Actual configured firewall rules, once exported
- `01-architecture/firewall-and-routing.md` — Validation checklist and traffic categories this document assumes
- `01-architecture/infrastructure-dependencies.md` — Source of the application-level dependencies behind Section 4
- `01-architecture/trust-boundaries.md` — Security rationale for each boundary assumed here