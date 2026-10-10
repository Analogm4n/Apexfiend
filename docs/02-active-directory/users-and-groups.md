# Apexfiend — Users and Groups

## 1. Purpose

This document records the verified security groups and user/computer/service-account memberships relevant to the Apexfiend attack-path design, as retrieved from `DC02.corp.apexfiend.lab` using `Get-ADGroup`, `Get-ADGroupMember`, and `Get-ADServiceAccount`.

This is not a complete domain user census. It records the identities and groups that are relevant to the documented infrastructure and attack-path design; built-in Active Directory groups are referenced only where they hold a narratively or technically relevant member.

## 2. Built-In Groups of Note

Standard built-in security groups (`Administrators`, `Account Operators`, `Backup Operators`, `Print Operators`, `Server Operators`, `Key Admins`, `DnsAdmins`, and others) exist with their default configuration and are not individually documented here, with one exception:

### Domain Admins

| Member | Type | SID suffix | Notes |
|---|---|---|---|
| `Administrator` | Built-in user | `-500` | Default domain Administrator account |
| `admin` | User | `-1106` | Lab-setup account created to configure servers and workstations during deployment; administrative noise, no narrative role |
| `m.alvarez` | User | `-1157` | Jenkins / CA01 Platform Administrator — the domain-administrative identity targeted by `attack-path.md` Stage 14 |

## 3. Custom Security Groups

The following custom (`GG-*` and `PKI-Operations`) global security groups exist in `OU=Groups`. Membership has been confirmed for the groups relevant to the documented attack path; the remainder are listed for completeness and flagged as not yet queried.

| Group | Description (as configured) | Membership |
|---|---|---|
| `GG-Infrastructure-Automation` | — | Confirmed: `svc_automation$` (sole member) |
| `GG-Jenkins-Hosts` | — | Confirmed: `JENKINS$`, `JENKINS01$` |
| `GG-HelpDesk-L1` | — | Confirmed: `l.sanchez`, `m.diaz`, `l.jimenez`, `m.romero` |
| `GG-HelpDesk-L2` | — | Confirmed: `j.gonzalez`, `s.lopez`, `m.rodriguez`, `m.garrido`, `j.sanz`, `v.arana` |
| `GG-WebOperations` | — | Confirmed: `d.garcia`, `d.ramirez` |
| `PKI-Operations` | — | Confirmed: includes `j.derek` |
| `GG-WebOperations-Support` | "Support and maintain user authentication and workstation..." (description truncated at capture) | Confirmed: includes `m.rodriguez` (Miguel) |
| `GG-Platform-Operations` | — | Not yet queried |
| `GG-Jenkins-Operators` | — | Not yet queried |
| `GG-Automation-Admins` | — | Not yet queried |
| `GG-Endpoint-Maintenance` | "Endpoint Maintenance Operators" | Not yet queried |
| `GG-HelpDesk-Lead` | — | Not yet queried |
| `GG-WebDev` | — | Not yet queried |
| `GG-WebOpsAdmin` | — | Not yet queried |

`GG-WebOperations-Support` is **not** a noise group — it carries the Shadow Credentials delegation central to the Miguel → Daniel stage (see `delegated-permissions.md`, Section 3). `GG-WebOpsAdmin` and `GG-WebDev` remain confirmed as lab realism/noise groups. `GG-WebOperations` itself (the group whose members are the attack-path targets Daniel and Diego) remains the group of actual interest as a target, not as a rights-holder.

## 4. Relevant User Identities

| SamAccountName | Full name | OU | SID suffix | Role |
|---|---|---|---|---|
| `l.sanchez` | Laura Sanchez | `IT/HelpDesk/HelpDesk-L1` | `-1132` | Help Desk L1 — recent hire, entry point for the Shadow Credentials stage of the attack path |
| `m.diaz` | — | `IT/HelpDesk/HelpDesk-L1` | `-1134` | Help Desk L1 |
| `l.jimenez` | Lucia Jimenez | `IT/HelpDesk/HelpDesk-L1` | `-1150` | Help Desk L1 |
| `m.romero` | Marco Romero | `IT/HelpDesk/HelpDesk-L1` | `-1151` | Help Desk L1 |
| `j.gonzalez` | — | `IT/HelpDesk/HelpDesk-L2` | `-1135` | Help Desk L2 |
| `s.lopez` | Sergio Lopez | `IT/HelpDesk/HelpDesk-L2` | `-1140` | Help Desk L2 |
| `m.rodriguez` | Miguel Rodriguez | `IT/HelpDesk/HelpDesk-L2` | `-1147` | Help Desk L2 — reached via Shadow Credentials against Laura's delegated rights; the deactivated-employee character in the attack-path design |
| `m.garrido` | Manuel Garrido | `IT/HelpDesk/HelpDesk-L2` | `-1152` | Help Desk L2 |
| `j.sanz` | Julian Sanz | `IT/HelpDesk/HelpDesk-L2` | `-1153` | Help Desk L2 |
| `v.arana` | Viviana Arana | `IT/HelpDesk/HelpDesk-L2` | `-1154` | Help Desk L2 |
| `d.garcia` | Daniel Garcia | `IT/ApplicationSupport/WebOperations` | `-1136` | Web Developer / DevOps Engineer — associated with WK02, MSSQL01, WEB02 |
| `d.ramirez` | Diego Ramirez | `IT/ApplicationSupport/WebOperations` | `-1141` | Web Operations |
| `m.ortega` | Marcos Ortega | `IT` (directly) | `-1112` | Jenkins Maintenance and Monitoring Operator — associated with WK04 |
| `m.alvarez` | — | `IT` | `-1157` | Jenkins / CA01 Platform Administrator; Domain Admin |
| `j.derek` | Johan Derek | `IT` (directly) | `-1610` | PKI Compliance Auditor — associated with WK03; confirmed member of `PKI-Operations` |

Full names for accounts shown only by SamAccountName (`m.diaz`, `j.gonzalez`) have not been separately queried; the user will add any missing full names directly when preparing the final repository copy.

## 5. Computer Objects

| SamAccountName | OU | SID suffix | Notes |
|---|---|---|---|
| `CA01$` | `corp-infrastructure` (directly) | `-1105` | Enterprise CA; `msDS-AllowedToActOnBehalfOfOtherIdentity` currently empty (no RBCD configured) |
| `JENKINS$` | `corp-infrastructure/Automation` | `-1613` | Jenkins-related computer object |
| `JENKINS01$` | `corp-infrastructure/Automation` | `-1127` | Jenkins controller host |

## 6. Service Accounts

### svc_automation$ (gMSA)

```text
DistinguishedName:                            CN=svc_automation,CN=Managed Service Accounts,DC=corp,DC=apexfiend,DC=lab
ObjectClass:                                   msDS-GroupManagedServiceAccount
SamAccountName:                                svc_automation$
SID suffix:                                    -1125
PrincipalsAllowedToRetrieveManagedPassword:    GG-Jenkins-Hosts
```

Only `JENKINS$` and `JENKINS01$` (the members of `GG-Jenkins-Hosts`) are authorized to retrieve this gMSA's managed password — consistent with the attack-path design, in which the gMSA's Kerberos material is obtained from the Jenkins process on `JENKINS01`.

`svc_automation$` is the sole member of `GG-Infrastructure-Automation` (see Section 3), which directly ties this service account to the delegated permissions documented in `delegated-permissions.md`.

An earlier capture of this output appeared to show two `svc_automation` entries; this was confirmed as a copy/paste artifact. There is exactly one `svc_automation$` gMSA in the domain.

### svc_reporting

Referenced in `01-architecture/system-inventory.md` as the reporting-database service account. Not yet confirmed against live AD (object type, OU placement, and group memberships are still pending verification).

## 7. Validation Status

| Item | Status |
|---|---|
| Domain Admins membership | Verified |
| `GG-Infrastructure-Automation`, `GG-Jenkins-Hosts`, `GG-HelpDesk-L1`, `GG-HelpDesk-L2`, `GG-WebOperations` membership | Verified |
| `svc_automation$` gMSA configuration and password-retrieval principals | Verified |
| `j.derek` OU placement and `PKI-Operations` membership | Verified |
| `m.rodriguez` (Miguel) membership in `GG-WebOperations-Support` | Verified |
| Remaining custom groups (`GG-Platform-Operations`, `GG-Jenkins-Operators`, `GG-Automation-Admins`, `GG-Endpoint-Maintenance`, `GG-HelpDesk-Lead`, `GG-WebDev`, `GG-WebOpsAdmin`) | Not yet queried |
| `svc_reporting` AD configuration | Not yet queried |

## 8. Related Documentation

- `domains.md` — Forest and domain configuration
- `organizational-units.md` — OU structure referenced by this document
- `delegated-permissions.md` — ACEs held by the groups and accounts listed here
- `../00-overview/attack-path.md` — Attack-path stages involving these identities