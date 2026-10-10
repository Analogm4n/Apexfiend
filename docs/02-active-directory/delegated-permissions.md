# Apexfiend — Delegated Permissions

## 1. Purpose

This document records delegated Active Directory permissions (ACEs) that are significant to the Apexfiend attack-path design, as retrieved from `DC02.corp.apexfiend.lab` using `dsacls` and the PowerShell `Get-Acl` AD provider.

Only delegations with a confirmed, documented security implication are recorded here. Where a designed attack-path step depends on a permission that has not yet been queried against the live environment, this is stated explicitly rather than assumed from the design.

## 2. corp-infrastructure — WriteDacl Delegation

```text
Object:    OU=corp-infrastructure,DC=corp,DC=apexfiend,DC=lab
Owner:     CORP\Domain Admins
```

### 2.1 Relevant ACE

```text
Trustee:                CORP\GG-Infrastructure-Automation
Access type:             Allow
Rights (on the OU):      ReadProperty, WriteProperty, WriteDacl, CreateChild
Inheritance:              Inherited to child computer objects as:
                              WriteProperty, ReadProperty (and effectively
                              WriteDacl via the OU-level grant)
```

This ACE is not inherited from a parent object — it is explicitly configured directly on `OU=corp-infrastructure`.

### 2.2 Security Implication

`GG-Infrastructure-Automation` is a global security group whose **sole member is `svc_automation$`** (see `users-and-groups.md`, Section 3). This means `svc_automation$` directly holds `WriteDacl` on `OU=corp-infrastructure`, and that permission is inherited down to every computer object placed directly in that OU.

`CA01$` is located directly in `OU=corp-infrastructure` (confirmed in `organizational-units.md`, Section 4) — not in a nested sub-OU such as `Critical-Servers`. As a result, `svc_automation$` can modify the DACL of the `CA01$` computer object itself. In practice, this means `svc_automation$` can grant itself (or another principal it controls) `GenericWrite` or an equivalent right directly on `CA01$`, which is sufficient to write to `msDS-AllowedToActOnBehalfOfOtherIdentity` and configure Resource-Based Constrained Delegation (RBCD) against `CA01$`.

This is the concrete, verified mechanism behind `attack-path.md` Stages 10–12:

```text
svc_automation$  (sole member of GG-Infrastructure-Automation)
      │
      ▼
WriteDacl on OU=corp-infrastructure  (confirmed ACE)
      │
      ▼
CA01$  (confirmed: located directly in this OU)
      │
      ▼
Self-granted rights → RBCD configuration on CA01$
```

No part of this chain currently exists in a pre-configured, exploited state — `CA01$`'s `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute is confirmed empty (see `users-and-groups.md`, Section 5). The *permission* that makes the escalation possible is verified; the escalation itself is a documented attack-path objective, not a pre-existing condition.

### 2.3 Other Notable Entries on This OU

The full ACL also includes standard/expected entries not specific to the lab's attack-path design:

- `CORP\Domain Admins` — Full Control (object owner)
- `APEXFIEND\Enterprise Admins` — Full Control (inherited from the forest root)
- `BUILTIN\Administrators` — broad special access (inherited)
- `BUILTIN\Account Operators` — `CreateChild`/`DeleteChild` for `user`, `computer`, `group`, and `inetOrgPerson` objects (default delegation)
- `APEXFIEND\Enterprise Key Admins` / `CORP\Key Admins` — `ReadProperty`/`WriteProperty` on `msDS-KeyCredentialLink` (inherited; this is the standard Windows Hello for Business / Shadow Credentials-relevant delegation built into the forest's default schema permissions, not a lab-specific grant)
- `NT AUTHORITY\SELF` — property self-write rights for `msDS-AllowedToActOnBehalfOfOtherIdentity` and related attributes (default computer self-permissions)

These are recorded for completeness; they reflect default Active Directory delegation present in any Windows2016-mode forest, not an intentional lab misconfiguration.

## 3. Help Desk Escalation Chain

Unlike the `corp-infrastructure` delegation, these ACEs are configured at the **OU** level — `OU=HelpDesk-L2` and `OU=WebOperations` — not on individual user objects or on the `GG-WebOperations` group object itself. Querying the group object directly (an earlier pass) returned no custom ACEs; the actual delegation only became visible once the correct OUs were queried.

### 3.1 OU=HelpDesk-L2 — Laura → Miguel

```text
Object: OU=HelpDesk-L2,OU=HelpDesk,OU=IT,DC=corp,DC=apexfiend,DC=lab
```

| Trustee | Right | Notes |
|---|---|---|
| `CORP\GG-HelpDesk-L1` | `ExtendedRight` — `00299570-246d-11d0-a768-00aa006e0529` (**Reset Password**) | Confirmed |
| `CORP\GG-HelpDesk-L1` | `WriteProperty` on several attribute/property-set GUIDs (`bf967a0a-...`, `28630ebf-...`, `bf967a68-...`) | Includes account-state attributes (enable/disable); individual GUIDs not all resolved to names |
| `CORP\GG-HelpDesk-L2` | `GenericAll` on this OU | Self-administration of their own tier; not externally relevant |

**`l.sanchez` (Laura) is the sole relevant member of `GG-HelpDesk-L1`** with standing to use this delegation against this OU. `m.rodriguez` (Miguel) is a member of `GG-HelpDesk-L2`, whose user object lives inside this OU and inherits these ACEs.

**Security implication:** Laura can enable and set the password of any disabled account in `OU=HelpDesk-L2` — specifically Miguel's dormant, ex-employee account. Because the account is not in active use, this action does not alert a live user or interrupt anyone's access; it is the quieter of the two privilege-escalation options available to Laura in the lab's design.

### 3.2 OU=WebOperations — Miguel → Daniel

```text
Object: OU=WebOperations,OU=ApplicationSupport,OU=IT,DC=corp,DC=apexfiend,DC=lab
```

| Trustee | Right | Notes |
|---|---|---|
| `CORP\GG-HelpDesk-L2` | `ExtendedRight` — `00299570-246d-11d0-a768-00aa006e0529` (**Reset Password**) | Confirmed; available but intentionally the noisier option |
| `CORP\GG-HelpDesk-L2` | `WriteProperty` on several additional attribute/property-set GUIDs (`bf967a0a-...`, `bf96794f-...`, `bf967950-...`, `bf967a68-...`, `be9ef6ee-...`) | Individual GUIDs not all resolved to names |
| `CORP\GG-WebOperations-Support` | `WriteProperty` — `5b47d60f-6090-40b2-9f37-2a4de88f3063` (**msDS-KeyCredentialLink**) | Confirmed — this is the Shadow Credentials-enabling ACE |

**`m.rodriguez` (Miguel) is confirmed as a member of both `GG-HelpDesk-L2` and `GG-WebOperations-Support`.** This means he holds *both* options against this OU:

- **Reset Password** via `GG-HelpDesk-L2` — technically available, but `d.garcia` (Daniel) is an actively-used account; resetting his password would immediately deny him access and alert him.
- **`msDS-KeyCredentialLink` write** via `GG-WebOperations-Support` — enables a Shadow Credentials attack, authenticating as Daniel via a certificate without ever touching his password or interrupting his session.

**Security implication — intentional OPSEC design:** the lab deliberately grants Miguel both the loud and the quiet option against the same target. The designed, tested attack path uses the `msDS-KeyCredentialLink` route specifically because Daniel's account is in active daily use; the Reset Password route exists in the environment as the "wrong" (detectable) choice a less careful operator might take.

### 3.3 Chain Summary

```text
l.sanchez (GG-HelpDesk-L1)
      │  ExtendedRight: Reset Password  (OU=HelpDesk-L2)
      │  — used: Miguel's account is dormant, low detection risk
      ▼
m.rodriguez (GG-HelpDesk-L2, GG-WebOperations-Support)
      │
      ├─ ExtendedRight: Reset Password (OU=WebOperations)      ─── available, NOT used (noisy)
      │
      └─ WriteProperty: msDS-KeyCredentialLink (OU=WebOperations) ─── used (stealthy)
      ▼
d.garcia (Daniel) — Shadow Credentials authentication, no password change, no interruption
```

This chain has been confirmed by the user as fully configured and tested end-to-end in the live lab.

## 4. Pending

- **`j.derek` / `PKI-Operations`** — the read-only ACE on CA01's certificate templates that this group is expected to carry has not yet been queried. This belongs more naturally in `06-pki/` and will be captured there.
- Several attribute-level `WriteProperty` GUIDs on both OUs above were not individually resolved to attribute names (only the two attack-relevant rights — Reset Password and `msDS-KeyCredentialLink` — were confidently identified). These can be resolved later against the schema partition if a complete attribute-level inventory is needed.

## 5. Validation Status

| Item | Status |
|---|---|
| `corp-infrastructure` OU — `GG-Infrastructure-Automation` WriteDacl ACE | Verified |
| `svc_automation$` sole membership of `GG-Infrastructure-Automation` | Verified |
| `CA01$` placement directly under `corp-infrastructure` | Verified |
| `CA01$` `msDS-AllowedToActOnBehalfOfOtherIdentity` is currently empty | Verified |
| `OU=HelpDesk-L2` — `GG-HelpDesk-L1` Reset Password delegation | Verified |
| `OU=WebOperations` — `GG-HelpDesk-L2` Reset Password delegation | Verified |
| `OU=WebOperations` — `GG-WebOperations-Support` `msDS-KeyCredentialLink` delegation | Verified |
| Full Help Desk escalation chain (Laura → Miguel → Daniel) | Verified and tested end-to-end by the user |
| `j.derek` / `PKI-Operations` certificate-template read delegation | Not yet queried (deferred to `06-pki/`) |

## 6. Related Documentation

- `organizational-units.md` — Structure of `OU=corp-infrastructure`
- `users-and-groups.md` — `GG-Infrastructure-Automation` and `svc_automation$` membership
- `acl-design.md` — Synthesis of this document into the broader security narrative
- `../00-overview/attack-path.md` — Stages 10–12 (WriteDACL / RBCD) rely on the ACE documented here