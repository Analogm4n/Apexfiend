# Apexfiend — Domains

## 1. Purpose

This document records the verified Active Directory forest and domain configuration for Apexfiend, as retrieved directly from `DC02.corp.apexfiend.lab`.

Unlike `docs/00-overview/` and `docs/01-architecture/`, which describe the intended design, this document and the rest of `02-active-directory/` record **confirmed configuration**, captured with native AD tooling (`Get-ADForest`, `Get-ADDomain`, `Get-ADTrust`, `nltest`). Where a detail has not yet been verified, it is marked as such rather than assumed.

## 2. Forest Overview

```text
Forest name:            apexfiend.lab
Forest mode:            Windows2016Forest
Root domain:             apexfiend.lab
Domains:                 apexfiend.lab, corp.apexfiend.lab
Sites:                   Default-First-Site-Name (single site)
Schema Master:            DC01.apexfiend.lab
Domain Naming Master:     DC01.apexfiend.lab
Global Catalogs:          DC01.apexfiend.lab, DC02.corp.apexfiend.lab, DC03.corp.apexfiend.lab
UPN Suffixes:             apexfiend.lab
Cross-forest references:  none
```

All three domain controllers are configured as Global Catalog servers, including DC01 in the forest root. The forest uses a single Active Directory site; there is no multi-site replication topology to consider.

## 3. Root Domain — apexfiend.lab

The root domain is hosted on a single domain controller:

| Attribute | Value |
|---|---|
| DNS name | `apexfiend.lab` |
| Domain controller | `DC01.apexfiend.lab` (`192.168.10.10`) |
| Role | Forest root; holds Schema Master and Domain Naming Master FSMO roles |

Detailed configuration of the root domain (OUs, users, groups) has not yet been queried, since the lab's identity-related activity is concentrated in the corporate child domain. This section should be expanded if root-domain objects become relevant to a documented attack path (for example, the forest-root compromise described in `attack-path.md` Stage 15).

## 4. Child Domain — corp.apexfiend.lab

```text
DNS name:                  corp.apexfiend.lab
NetBIOS name:               CORP
Domain mode:                 Windows2016Domain
Domain SID:                  S-1-5-21-3364994804-235286439-3869108742
Parent domain:                apexfiend.lab
PDC Emulator:                 DC02.corp.apexfiend.lab
RID Master:                   DC02.corp.apexfiend.lab
Infrastructure Master:         DC02.corp.apexfiend.lab
Replica directory servers:     DC02.corp.apexfiend.lab, DC03.corp.apexfiend.lab
Computers container:           CN=Computers,DC=corp,DC=apexfiend,DC=lab
Users container:               CN=Users,DC=corp,DC=apexfiend,DC=lab
Domain Controllers container:  OU=Domain Controllers,DC=corp,DC=apexfiend,DC=lab
```

All three domain-level FSMO roles present in the child domain (PDC Emulator, RID Master, Infrastructure Master) are held by DC02. DC03 operates purely as an additional domain controller and Global Catalog, with no FSMO roles.

The corporate child domain holds effectively all of the lab's identity, group, and OU structure relevant to the documented attack paths. See `organizational-units.md` and `users-and-groups.md` for the object-level detail.

## 5. Inter-Domain Trust

```text
Trust source:              corp.apexfiend.lab
Trust target:               apexfiend.lab
Direction:                   BiDirectional
Trust type:                  Uplevel (intra-forest)
IntraForest:                  True
ForestTransitive:             False
TGT Delegation:               False
Selective Authentication:     False
SID Filtering (Quarantined):  False
SID Filtering (Forest-Aware): False
```

This is the implicit, automatically-created parent-child trust that exists between any two domains in the same Active Directory forest; it was not manually configured.

**Security relevance:** `SIDFilteringQuarantined` is `False`. SID filtering quarantine is a mitigation that strips SID history values arriving from an external or cross-forest trust; it does **not** apply to intra-forest parent-child trusts by default, since SID history is the normal mechanism Microsoft uses for intra-forest domain migrations. This is expected default behavior, not a lab-specific misconfiguration — but it is precisely what makes the forest-root compromise described in `attack-path.md` Stage 15 (SID History / ExtraSid abuse from `DC02$` to `DC01$`) viable: a forged or legitimately-issued ticket carrying a root-domain Enterprise Admins SID in its SID history will be honored by the root domain, because nothing in this trust configuration filters it out.

## 6. Replication

Replication between DC01, DC02, and DC03 uses standard RPC-based intra-site replication under the single `Default-First-Site-Name` site. Replication topology and health should be reverified periodically with `repadmin /showrepl` as the lab evolves; transient replication errors observed during setup were not treated as a documented finding, as they were resolved without a configuration change.

## 7. Validation Status

| Item | Status |
|---|---|
| Forest structure (`Get-ADForest`) | Verified |
| Child domain configuration (`Get-ADDomain`) | Verified |
| FSMO role placement | Verified |
| Inter-domain trust configuration (`Get-ADTrust`, `nltest`) | Verified |
| Root domain (`apexfiend.lab`) object-level detail | Not yet queried |
| Password and lockout policy | Not yet queried — requires `Get-ADDefaultDomainPasswordPolicy`, not `Get-ADDomain *Policy*` (the latter only returns linked GPO references) |

## 8. Related Documentation

- `organizational-units.md` — OU structure of the corporate domain
- `users-and-groups.md` — Users, groups, and the gMSA
- `delegated-permissions.md` — ACEs delegated within the domain
- `kerberos.md` — Delegation and ticket-related configuration
- `../00-overview/attack-path.md` — Stage 15 relies on the trust configuration described here