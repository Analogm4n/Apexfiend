# Apexfiend — ACL Design

## 1. Purpose

This document synthesizes the delegated permissions recorded in `delegated-permissions.md` and `kerberos.md` into the two privilege-escalation narratives they enable. It does not introduce new queries against the live environment — it explains *why* the verified ACEs matter and how they chain together.

Two independent, verified delegation chains exist in the lab. Both originate from a routine, legitimate-looking delegation and end in control over a more sensitive identity or system.

## 2. Chain 1 — Infrastructure Automation to Certificate Authority

### 2.1 The Delegation

```text
svc_automation$  (gMSA; sole member of GG-Infrastructure-Automation)
      │
      ▼
WriteDacl on OU=corp-infrastructure   (verified ACE, explicit on the OU)
      │
      ▼
CA01$   (verified: located directly inside this OU, not in a nested sub-OU)
```

### 2.2 Why It Matters

A `WriteDacl` grant on an OU is, by itself, a routine administrative delegation — it lets an automation identity manage access to the servers it is responsible for monitoring. The security impact only appears once two further facts are combined:

1. **`CA01$`'s placement.** Because the Enterprise CA's computer object sits directly in `OU=corp-infrastructure` rather than a nested sub-OU such as `Critical-Servers`, it inherits this `WriteDacl` grant like any other computer object in the OU. A more defensively-designed OU structure might isolate the CA's computer object under a sub-OU with a narrower, independently-managed ACL — but that is not how this environment is configured.
2. **`svc_automation$`'s narrow membership.** The gMSA is the *only* member of `GG-Infrastructure-Automation`. There is no ambiguity about which identity can exercise this right — compromising the gMSA's Kerberos material (the documented objective of `attack-path.md` Stages 8–9, via the Jenkins process on `JENKINS01`) is sufficient on its own to reach this delegation.

`WriteDacl` on a computer object allows the holder to grant itself further rights on that object — specifically enough to write `msDS-AllowedToActOnBehalfOfOtherIdentity`, configuring Resource-Based Constrained Delegation against `CA01$`. This is `attack-path.md` Stage 12. `kerberos.md`, Section 2 confirms that attribute is currently empty — the permission is live, but the RBCD configuration itself has not been pre-staged.

### 2.3 Design Observation

This chain is an example of **indirect privilege** rather than a direct misconfiguration: no single ACE looks dangerous in isolation. `WriteDacl` on a server OU is a common real-world delegation for teams that manage server onboarding. The risk only materializes because of where one specific computer object happens to live within that OU — a realistic class of finding in real environments, where OU placement drifts over time independent of the original delegation's intent.

## 3. Chain 2 — Help Desk Tier Escalation

### 3.1 The Delegation

```text
l.sanchez (Laura) — GG-HelpDesk-L1
      │
      ▼
Reset Password (ExtendedRight) on OU=HelpDesk-L2    (verified)
      │
      ▼
m.rodriguez (Miguel) — GG-HelpDesk-L2, GG-WebOperations-Support
      │
      ├── Reset Password (ExtendedRight) on OU=WebOperations     (verified; available, not used)
      │
      └── msDS-KeyCredentialLink WriteProperty on OU=WebOperations   (verified; used)
      │
      ▼
d.garcia (Daniel) — GG-WebOperations
```

### 3.2 Why It Matters

Both delegations in this chain follow a common, defensible-looking pattern: a higher support tier is granted account-management rights over the tier below it, scoped by OU. Individually, "L1 can reset L2's password" and "L2 can manage WebOperations accounts" are unremarkable help-desk delegations.

The chain becomes an attack path for two reasons:

1. **It runs in an unexpected direction.** `GG-HelpDesk-L1` holding Reset Password over `GG-HelpDesk-L2` — a *higher* tier — is the reverse of the usual intuition that senior tiers manage junior ones. A permissions review that only checks "can L2 manage L1" would miss this.
2. **Miguel's account is dormant.** `m.rodriguez` belongs to a deactivated former-employee identity. Activating and setting an initial password on an inactive account does not interrupt any live user's access and is comparatively low-risk to detect, compared to resetting the password of someone actively using their account.

### 3.3 The OPSEC Decision Point

Against `OU=WebOperations`, Miguel (as a member of both `GG-HelpDesk-L2` and `GG-WebOperations-Support`) holds two independent rights over the same target:

| Option | Mechanism | Operational cost |
|---|---|---|
| Reset Password | `GG-HelpDesk-L2` → `ExtendedRight` on `OU=WebOperations` | Immediately locks Daniel out of his own account; he notices and reports it the moment he tries to log in |
| `msDS-KeyCredentialLink` write (Shadow Credentials) | `GG-WebOperations-Support` → `WriteProperty` on `OU=WebOperations` | Authenticates as Daniel via a certificate derived from an attacker-supplied key; Daniel's own password and session are never touched |

Daniel's account is in active daily use — unlike Miguel's. The designed and tested attack path uses the Shadow Credentials option specifically for this reason. This is an intentional feature of the lab: a less careful operator has the noisier option available and might reach for it reflexively, while a more deliberate one recognizes that the target's account activity level should determine which available right to exercise.

### 3.4 Design Observation

This chain demonstrates that **having a right does not mean it should be exercised** — the lab rewards recognizing which of two technically equivalent escalation paths is operationally safer, rather than simply rewarding privilege discovery. This mirrors real engagement tradecraft, where a redundant or "louder" path is frequently available alongside a quieter one, and the choice between them is itself a skill being assessed.

## 4. Common Pattern Across Both Chains

Both verified chains share a structure worth naming explicitly:

```text
Narrow, single-purpose group membership
      +
A delegated right that looks routine in isolation
      +
An object placement or account-state detail that is easy to overlook
      =
A privilege-escalation path
```

In Chain 1, the overlooked detail is `CA01$`'s OU placement. In Chain 2, it is the direction of the Help Desk delegation and the live/dormant status of the target accounts. Neither chain depends on a software vulnerability, a weak password, or a classic misconfiguration like `GenericAll` granted carelessly — both rely entirely on legitimate-looking Active Directory delegation, which is the central thesis of the Apexfiend project as stated in `../00-overview/objectives.md`.

## 5. Status

Both chains documented here are **verified and tested end-to-end** in the live environment, per direct confirmation from the lab operator — not merely designed. This is a stronger validation status than most of `../00-overview/attack-path.md`'s current table reflects for the corresponding stages; that table should be revisited to mark the Help Desk escalation and `WriteDacl`/OU-placement stages as tested rather than designed, once a full pass over `attack-path.md`'s status column is convenient.

## 6. Related Documentation

- `organizational-units.md` — OU structure referenced by both chains
- `users-and-groups.md` — Group membership underlying both chains
- `delegated-permissions.md` — The individual ACEs this document synthesizes
- `kerberos.md` — RBCD precondition state for Chain 1
- `../00-overview/attack-path.md` — Stages 10–12 (Chain 1) and the early Help Desk stages (Chain 2)