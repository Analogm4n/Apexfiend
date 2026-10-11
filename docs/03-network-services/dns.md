# Apexfiend — DNS

## 1. Purpose

This document records the verified DNS configuration of `corp.apexfiend.lab`, as retrieved from `DC02.corp.apexfiend.lab` using `Get-DnsServerZone`, `Get-DnsServerForwarder`, and `Get-DnsServerResourceRecord`.

**IP addresses in this document are the authoritative, current values**, confirmed directly from the live DNS server. Where they differ from addresses recorded earlier in `01-architecture/` or `02-active-directory/`, this document is correct; those documents have not been retroactively corrected and should be treated as superseded on this point until the operator updates them manually.

## 2. DNS Zones

```powershell
Get-DnsServerZone
```

| Zone | Type | Auto-created | AD-integrated | Reverse lookup |
|---|---|---|---|---|
| `_msdcs.apexfiend.lab` | Primary | No | Yes | No |
| `0.in-addr.arpa` | Primary | Yes | No | Yes |
| `127.in-addr.arpa` | Primary | Yes | No | Yes |
| `255.in-addr.arpa` | Primary | Yes | No | Yes |
| `apexfiend.lab` | Primary | No | Yes | No |
| `corp.apexfiend.lab` | Primary | No | Yes | No |
| `TrustAnchors` | Primary | No | Yes | No |

All zones are DNS-integrated into Active Directory, with the exception of the three default/auto-created reverse-lookup zones (standard placeholders for `0.0.0.0`, `127.0.0.1`, and broadcast addresses).

**No reverse-lookup zone exists for the lab's internal ranges** (`172.16.0.0/16`, `192.168.10.0/24`, etc.) — PTR-based (reverse DNS) resolution is not currently configured anywhere in the corporate network.

## 3. Forwarding

```powershell
Get-DnsServerForwarder
```

```text
IPAddress:       192.168.10.10   (DC01.apexfiend.lab)
UseRootHint:      True
Timeout:          3 seconds
EnableReordering: True
```

DC02 forwards unresolved queries to `DC01` (the root-domain controller) rather than directly to an external resolver, falling back to root hints if DC01 does not respond. This means external name resolution for the corporate domain depends on DC01's own upstream configuration, which has not yet been separately queried.

## 4. corp.apexfiend.lab — A Records

```powershell
Get-DnsServerResourceRecord -ZoneName "corp.apexfiend.lab" -RRType A
```

| Hostname | IP Address | Notes |
|---|---|---|
| `@` (zone apex) | `172.16.20.100`, `172.16.20.101` | Standard AD domain-apex records (DC02, DC03) |
| `CA01` | `172.16.50.100` | Enterprise CA |
| `dc02` | `172.16.20.100` | |
| `DC03` | `172.16.20.101` | |
| `DomainDnsZones` | `172.16.20.100`, `172.16.20.101` | Standard AD-integrated DNS service record |
| `GITEA01` | `172.16.40.100` | |
| `JENKINS` | `172.16.50.101` | Legacy record — same host as `JENKINS01`, predating a rename (confirmed by the operator) |
| `JENKINS01` | `172.16.50.102` | Current Jenkins controller hostname |
| `MSSQL01` | `172.16.40.104` | |
| `MSSQL02` | `172.16.40.105` | |
| `web03` | `172.16.40.103` | |
| `WK01` | `172.16.60.104` | |
| `WK02` | `172.16.60.105` | |
| `WK03` | `172.16.60.106` | |
| `WK04` | `172.16.60.107` | |

**`JENKINS01` is hosted in VLAN 50** (`172.16.50.0/24`) alongside `CA01` — not in a separate automation-specific VLAN. This places the Jenkins controller and the Enterprise CA in the same network segment, which is directly relevant to `00-overview/attack-path.md`'s later stages (the WriteDACL/RBCD chain against `CA01$` does not require crossing a VLAN boundary from `JENKINS01`).

## 5. MAIL01

`MAIL01` does not have an A record in `corp.apexfiend.lab` — it is registered in the **root** zone, `apexfiend.lab`, consistent with the mail environment using `@apexfiend.lab` addresses while AD identities use `@corp.apexfiend.lab` (as already noted in `01-architecture/infrastructure-dependencies.md`, Section 8).

```text
MAIL01: 172.16.40.106   (static; confirmed by the operator, not independently queried via Get-DnsServerResourceRecord against the apexfiend.lab zone)
```

## 6. Validation Status

| Item | Status |
|---|---|
| Zone list and configuration | Verified |
| Forwarder configuration | Verified |
| `corp.apexfiend.lab` A records | Verified |
| `MAIL01` address and zone placement | Operator-confirmed; not independently queried against the `apexfiend.lab` zone |
| Reverse-lookup (PTR) configuration for internal ranges | Verified absent — not configured |
| DC01's own forwarder/upstream DNS configuration | Not yet queried |

## 7. Related Documentation

- `pfsense.md` — Network segments these hosts belong to
- `02-active-directory/domains.md` — Forest and domain structure these zones serve
- `01-architecture/system-inventory.md` — Superseded on IP addresses by this document; not yet updated