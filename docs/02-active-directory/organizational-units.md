# Apexfiend — Organizational Units

## 1. Purpose

This document records the verified Organizational Unit (OU) structure of `corp.apexfiend.lab`, as retrieved from `DC02.corp.apexfiend.lab` using `Get-ADOrganizationalUnit`.

The OU design separates **computer objects** (servers, automation hosts) from **user objects** (staff organized by department and role), and includes the specific OU whose delegated permissions are central to the lab's documented attack path. Delegated ACEs on these OUs are detailed separately in `delegated-permissions.md`; this document records structure and purpose only.

## 2. OU Tree

```text
corp.apexfiend.lab
├── corp-infrastructure                    (computer objects)
│   ├── Application-Servers
│   ├── Automation
│   ├── Critical-Servers
│   └── Database-Servers
├── Domain Controllers                     (default container)
├── Engineering
├── Finance
├── Groups
├── HR
├── IT                                     (user objects)
│   ├── ApplicationSupport
│   │   └── WebOperations
│   ├── Automation
│   ├── HelpDesk
│   │   ├── HelpDesk-L1
│   │   └── HelpDesk-L2
│   └── Infrastructure
└── Workstations
```

## 3. Design Pattern: Computers vs. Users

The two top-level OUs most relevant to the lab's attack-path design follow a consistent, intentional split:

- **`OU=corp-infrastructure`** holds **computer objects**, organized by server function (`Application-Servers`, `Automation`, `Critical-Servers`, `Database-Servers`).
- **`OU=IT`** holds **user objects**, organized by department/team structure (`ApplicationSupport`, `Automation`, `HelpDesk`, `Infrastructure`).

Both top-level OUs happen to contain a sub-OU named `Automation`; they are not the same object and do not hold the same kind of principal. `OU=Automation,OU=corp-infrastructure` holds automation-related computer accounts; `OU=Automation,OU=IT` holds the user accounts of staff working in automation.

## 4. corp-infrastructure

```text
DistinguishedName: OU=corp-infrastructure,DC=corp,DC=apexfiend,DC=lab
```

This OU and its sub-OUs hold the lab's server-class computer objects.

| Sub-OU | Purpose |
|---|---|
| `Application-Servers` | Application-tier server computer objects |
| `Automation` | Automation infrastructure computer objects (Jenkins hosts) |
| `Critical-Servers` | High-sensitivity server computer objects |
| `Database-Servers` | SQL Server computer objects |

**`CA01$` is located directly under `OU=corp-infrastructure`**, not under the `Critical-Servers` sub-OU. This placement is confirmed as intentional lab design, not an oversight.

`corp-infrastructure` carries a delegated ACE — held by `GG-Infrastructure-Automation` — granting `WriteDacl` and inherited `WriteProperty` on computer objects within the OU (see `delegated-permissions.md`). Because `CA01$` sits directly inside this OU rather than in a nested sub-OU, it is directly subject to that delegation. This is the specific ACE that enables the WriteDACL and RBCD stages of the documented attack path (`attack-path.md`, Stages 10–12).

`JENKINS$` and `JENKINS01$` are located under `OU=Automation,OU=corp-infrastructure` (confirmed via `Get-ADGroupMember "GG-Jenkins-Hosts"`).

## 5. Domain Controllers

```text
DistinguishedName: OU=Domain Controllers,DC=corp,DC=apexfiend,DC=lab
```

The default Active Directory container for domain controller computer objects (`DC02$`, `DC03$`). Not modified from its default configuration.

## 6. Engineering, Finance, HR

```text
OU=Engineering,DC=corp,DC=apexfiend,DC=lab
OU=Finance,DC=corp,DC=apexfiend,DC=lab
OU=HR,DC=corp,DC=apexfiend,DC=lab
```

Standard corporate departmental OUs. No user population or delegated permissions have been queried for these OUs yet; they exist to represent a realistic organizational structure and are not currently part of a documented attack path.

## 7. Groups

```text
DistinguishedName: OU=Groups,DC=corp,DC=apexfiend,DC=lab
```

Holds the domain's custom security groups (the `GG-*` and `PKI-Operations` groups). See `users-and-groups.md` for the full group inventory and membership.

## 8. IT

```text
DistinguishedName: OU=IT,DC=corp,DC=apexfiend,DC=lab
```

Holds the IT department's user accounts, organized into the following sub-OUs. `m.ortega` (Marcos Ortega) is located directly under `OU=IT`, not inside any of its sub-OUs.

| Sub-OU | Purpose |
|---|---|
| `ApplicationSupport/WebOperations` | Web Operations team users (`d.garcia`, `d.ramirez`) |
| `Automation` | Automation team user accounts |
| `HelpDesk/HelpDesk-L1` | Tier-1 Help Desk user accounts (`l.sanchez`, `m.diaz`, `l.jimenez`, `m.romero`) |
| `HelpDesk/HelpDesk-L2` | Tier-2 Help Desk user accounts (`j.gonzalez`, `s.lopez`, `m.rodriguez`, `m.garrido`, `j.sanz`, `v.arana`) |
| `Infrastructure` | Infrastructure team user accounts |

Membership detail for each relevant group is recorded in `users-and-groups.md`.

## 9. Workstations

```text
DistinguishedName: OU=Workstations,DC=corp,DC=apexfiend,DC=lab
```

Intended to hold the domain-joined workstation computer objects (`WK01$`–`WK05$`). Object-level confirmation of its contents is still pending.

## 10. Validation Status

| Item | Status |
|---|---|
| Full OU tree (`Get-ADOrganizationalUnit`) | Verified |
| `CA01$` placement under `corp-infrastructure` | Verified |
| `JENKINS$` / `JENKINS01$` placement under `corp-infrastructure/Automation` | Verified |
| `m.ortega` placement directly under `IT` | Verified |
| Contents of `Engineering`, `Finance`, `HR`, `Workstations` | Not yet queried |
| Linked GPOs per OU | Partially recorded; not yet analyzed for security impact |

## 11. Related Documentation

- `domains.md` — Forest and domain configuration
- `users-and-groups.md` — Users, groups, and gMSA membership
- `delegated-permissions.md` — ACEs delegated on `corp-infrastructure` and other OUs
- `../00-overview/attack-path.md` — Stages 10–12 depend on the `corp-infrastructure` OU structure described here