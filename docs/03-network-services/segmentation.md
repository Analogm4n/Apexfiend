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

## 5. Verified Findings (from `pfsense.md`)

The live firewall configuration has now been reviewed (see `pfsense.md`, Sections 6–7). Results differ by segment:

| Segment | Result |
|---|---|
| VLAN 10 → VLAN 20/40 only, blocked otherwise | **Verified — matches intended design** |
| VLAN 20 → all domain-joined segments, blocked otherwise | **Verified — matches intended design** |
| VLAN 60 (workstations) → restricted set, blocked otherwise | **Verified — matches intended design** |
| VPN clients restricted to VLAN 60 | **Verified — enforced via pushed OpenVPN route, not a firewall rule** |
| **VLAN 30 (DMZ) egress restricted to 80/443** | **Does not match — unrestricted egress currently configured; the restrictive rule exists but is disabled** |
| **VLAN 40 egress restricted to documented dependencies** | **Does not match — unrestricted egress currently configured; the restrictive rule exists but is disabled** |
| **VLAN 50 (CA01, JENKINS01) egress restricted to documented dependencies** | **Does not match — unrestricted egress currently configured; the restrictive rule exists but is disabled** |

The DMZ, VLAN 40, and VLAN 50 findings are genuine current-state misconfigurations relative to the design intent recorded in this document and in `01-architecture/trust-boundaries.md` — not an intentional attack-path feature. They are retained here as real findings; see `pfsense.md` for the underlying rule-by-rule detail.

## 6. Validation Status

| Rule | Status |
|---|---|
| VLAN 10 and VLAN 20 segmentation | Verified — matches design |
| VLAN 60 segmentation | Verified — matches design |
| VPN client restriction to VLAN 60 | Verified — matches design (route-based) |
| DMZ egress restricted to 80/443 | Verified — **does not match**; currently unrestricted |
| VLAN 40 egress restriction | Verified — **does not match**; currently unrestricted |
| VLAN 50 egress restriction | Verified — **does not match**; currently unrestricted |
| Remaining application-specific rules in Section 4 not covered above | Not yet individually tested |

## 7. Related Documentation

- `pfsense.md` — Verified firewall rules this section is derived from
- `dns.md` — DNS configuration for the hosts referenced throughout this document
- `vpn-config.md` — OpenVPN configuration detail
- `01-architecture/firewall-and-routing.md` — Validation checklist and traffic categories this document assumes
- `01-architecture/infrastructure-dependencies.md` — Source of the application-level dependencies behind Section 4
- `01-architecture/trust-boundaries.md` — Security rationale undermined by the Section 5 findings