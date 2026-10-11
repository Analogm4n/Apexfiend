# Apexfiend — pfSense

## 1. Purpose

This document records the verified firewall and routing configuration of the lab's pfSense gateway, exported from the live `config.xml`. It supersedes the intended/assumed segmentation model documented in `segmentation.md` wherever the two disagree — this document reflects what is actually configured, not what was designed.

Sensitive material present in the raw export (bcrypt password hashes, private keys for the OpenVPN CA, server certificate, and client certificates, and the TLS-auth static key) has been deliberately excluded from this document and should never be committed to the public repository in any form.

## 2. Platform

```text
Version:    pfSense 24.6
Hostname:   pfSense.home.arpa
```

## 3. Interfaces

| Interface | Physical | Network | Description |
|---|---|---|---|
| WAN | `em0` | DHCP (external) | Internet-facing, dynamic IP |
| LAN | `em1` | `192.168.1.1/24` | Default LAN; not part of the documented VLAN architecture |
| VLAN 10 | `em1.10` | `192.168.10.1/24` | Root Active Directory (DC01) |
| VLAN 20 | `em1.20` | `172.16.20.1/24` | Corporate Active Directory (DC02, DC03) |
| VLAN 30 | `em1.30` | `172.16.30.1/24` | DMZ (WEB01, WEB02) |
| VLAN 40 | `em1.40` | `172.16.40.1/24` | Corporate applications and data |
| VLAN 50 | `em1.50` | `172.16.50.1/24` | PKI / automation (CA01, Jenkins) |
| VLAN 60 | `em1.60` | `172.16.60.1/24` | Workstations |

A seventh VLAN (`VLAN 90`) exists in the live configuration as a testing/attacker-workstation network and is intentionally **not** part of the documented corporate environment — excluded from this and all other architecture documentation.

## 4. WAN NAT — SSH to WEB01

```text
Rule: WAN SSH → WEB01
Source:       10.0.100.12   (restricted to this single address)
Destination:  WAN interface, TCP 22
Target:       172.16.30.100 : 22   (WEB01)
```

This confirms `WEB01`'s address as `172.16.30.100`. Only the single source address `10.0.100.12` is permitted to reach WEB01 over SSH through the WAN — this is the external entry point underlying `00-overview/attack-path.md` Stage 1 and the documented SSH-agent-forwarding pivot from WEB01 onward. No other inbound NAT/port-forward rules exist.

## 5. OpenVPN Access Rule

```text
Interface: openvpn
Source:      any
Destination: any
Type:        pass
```

This rule permits all traffic from already-authenticated VPN clients at the firewall level. In practice, client reach is constrained by what the OpenVPN server pushes as a route — confirmed in `vpn-config.md` to be `172.16.60.0/24` only — rather than by this firewall rule itself. This **verifies** the "VPN clients restricted to VLAN 60" assumption in `segmentation.md`, Section 5, via the pushed-route mechanism rather than a firewall-level restriction.

A separate rule permits inbound UDP 1194 to the WAN interface (the OpenVPN listener itself).

## 6. Inter-VLAN Rules (Verified, by Interface)

Each VLAN's rules are evaluated top-down; the first match wins. Rules below are listed as they appear in the live configuration, not reordered.

### VLAN 10 (Root AD)

| Order | Rule |
|---|---|
| 1 | Pass → VLAN 40 |
| 2 | Pass → VLAN 20 |
| 3 | Block → any (final) |

**Verified:** correctly restricted — the final block is enabled.

### VLAN 20 (Corporate AD — DC02, DC03)

| Order | Rule |
|---|---|
| 1 | Pass → VLAN 40 |
| 2 | Pass → VLAN 90 (test network) |
| 3 | Pass → VLAN 60 |
| 4 | Pass → VLAN 50 |
| 5 | Pass → VLAN 20 (self) |
| 6 | Pass → VLAN 10 |
| 7 | Block → any (final) |

**Verified:** correctly restricted — domain controllers can reach every domain-joined segment plus themselves, and the final block is enabled. No rule permits VLAN 20 → VLAN 30 (DMZ).

### VLAN 30 (DMZ — WEB01, WEB02)

| Order | Rule |
|---|---|
| 1 | **Pass → any** (enabled) |
| 2 | Block → any (final) — **disabled** |

### VLAN 40 (Corporate Applications and Data)

| Order | Rule |
|---|---|
| 1 | **Pass → any** (enabled) |
| 2 | Pass → VLAN 30 (DMZ) |
| 3 | Pass → VLAN 20 — *"DC's communications"* |
| 4 | Pass → VLAN 90 — *"Kali testing"* |
| 5 | Pass → VLAN 40 (self) |
| 6 | Block → any (final) — **disabled** |

### VLAN 50 (PKI / Automation — CA01, JENKINS01)

| Order | Rule |
|---|---|
| 1 | **Pass → any** (enabled) |
| 2 | Pass → VLAN 10 |
| 3 | Pass → VLAN 60 |
| 4 | Pass → VLAN 90 (test) |
| 5 | Pass → VLAN 50 (self) |
| 6 | Pass → VLAN 20 |
| 7 | Block → any (final) — **disabled** |


### VLAN 60 (Workstations)

| Order | Rule |
|---|---|
| 1 | Pass → VLAN 40 |
| 2 | Pass from WAN → VLAN 60 |
| 3 | Pass → VLAN 50 |
| 4 | Pass → VLAN 90 (test) |
| 5 | Pass → VLAN 20 |
| 6 | Pass → VLAN 60 (self) |
| 7 | Block → any (final) — **enabled** |

**Verified:** correctly restricted. This is the only non-DC, non-root VLAN with no broad allow rule and an active final block — consistent with `segmentation.md`'s intended model.

## 7. Summary of Verified vs. Intended Segmentation

| Segment | Intended (per `segmentation.md`) | Verified (per live `pfsense.xml`) |
|---|---|---|
| VLAN 10 (Root AD) | Restricted to AD ports toward required segments | **Matches** — restricted as designed |
| VLAN 20 (Corp AD) | Restricted, reachable by all domain-joined segments | **Matches** — restricted as designed |
| VLAN 30 (DMZ) | Egress limited to TCP 80/443 only, no route to corporate segments | **Does not match** — unrestricted egress, final block disabled |
| VLAN 40 (Apps/Data) | Application-specific access only | **Does not match** — unrestricted egress, final block disabled |
| VLAN 50 (PKI/Automation) | AD + enrollment ports only | **Does not match** — unrestricted egress, final block disabled |
| VLAN 60 (Workstations) | Restricted to required services | **Matches** — restricted as designed |
| VPN clients | Restricted to VLAN 60 only | **Matches** — enforced via pushed route, not firewall rule |

This summary should be used to update `segmentation.md`'s validation status (see that document's Section 5) rather than duplicated there in full.

## 8. Validation Status

| Item | Status |
|---|---|
| Interface and VLAN assignments | Verified from live config |
| WAN NAT rule (SSH → WEB01) | Verified |
| OpenVPN firewall rule and pushed-route restriction | Verified |
| Inter-VLAN rule sets (all 6 corporate VLANs) | Verified from live config |
| DMZ, VLAN 40, VLAN 50 unrestricted-egress finding | Verified — real, current misconfiguration, not a design choice |

## 9. Related Documentation

- `dns.md` — DNS zones and records served from the corporate domain
- `vpn-config.md` — OpenVPN server configuration detail
- `../01-architecture/network-architecture.md` — Intended topology this configuration implements
- `../01-architecture/trust-boundaries.md` — Security rationale the DMZ/VLAN 40/VLAN 50 findings undermine
- `segmentation.md` — Intended segmentation model; Section 5 should be updated against Section 7 of this document