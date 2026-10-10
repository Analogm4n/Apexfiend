# Apexfiend — Kerberos

## 1. Purpose

This document records Kerberos-relevant configuration for `corp.apexfiend.lab` — delegation settings, account policy, and the trust properties that affect ticket handling — as retrieved from `DC02.corp.apexfiend.lab`. It complements `domains.md` (inter-domain trust) and `delegated-permissions.md` (the ACEs that enable the RBCD stage of the attack path).

## 2. Delegation Configuration

```powershell
Get-ADUser -Filter {TrustedForDelegation -eq $true -or TrustedToAuthForDelegation -eq $true} -Properties TrustedForDelegation,TrustedToAuthForDelegation,msDS-AllowedToDelegateTo
Get-ADComputer -Filter {TrustedForDelegation -eq $true -or TrustedToAuthForDelegation -eq $true} -Properties TrustedForDelegation,TrustedToAuthForDelegation,msDS-AllowedToActOnBehalfOfOtherIdentity
```

| Object | TrustedForDelegation | TrustedToAuthForDelegation | Notes |
|---|---|---|---|
| `DC02$` | True | False | Default behavior for domain controllers (unconstrained delegation is standard on DCs; not attacker-relevant) |
| `DC03$` | True | False | Same as above |

No user object matched the filter. No computer object other than the two domain controllers has any form of delegation (unconstrained or constrained) configured.

Specifically confirmed empty:

- `CA01$` — `msDS-AllowedToActOnBehalfOfOtherIdentity` is not set (no RBCD configured against it)
- `m.ortega` — `msDS-AllowedToDelegateTo` is not set (no constrained delegation configured)
- `svc_automation$` — no delegation configured

**This is the expected state prior to exploitation.** `attack-path.md` Stages 10–12 describe `svc_automation$` abusing its `WriteDacl` on `OU=corp-infrastructure` (verified in `delegated-permissions.md`, Section 2) to configure RBCD against `CA01$` — i.e., to populate `CA01$`'s `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute. That attribute being empty today confirms the attack has not been pre-staged; the permission that makes it possible has been verified independently.

## 3. Domain Account Policy

```powershell
Get-ADDefaultDomainPasswordPolicy
```

```text
ComplexityEnabled:             True
MinPasswordLength:              7
MaxPasswordAge:                 42.00:00:00  (42 days)
MinPasswordAge:                 1.00:00:00   (1 day)
PasswordHistoryCount:           24
ReversibleEncryptionEnabled:    False
LockoutDuration:                00:30:00  (30 minutes)
LockoutObservationWindow:       00:30:00  (30 minutes)
LockoutThreshold:                0
```

**`LockoutThreshold` is `0`**, meaning account lockout is effectively disabled — there is no limit on failed authentication attempts before a lockout would occur. `LockoutDuration` and `LockoutObservationWindow` are configured (30 minutes each), but they have no effect while the threshold itself is 0.

**Security relevance:** this permits unthrottled password-guessing attempts (online brute force or password spraying) against any domain account without risk of locking it out. Whether this is an intentional lab design choice (to support offline/online credential-attack practice) or an oversight has not yet been confirmed with the user; it is recorded here as observed configuration, not as a finding with an assumed cause.

## 4. Inter-Domain Trust and Kerberos

The `corp.apexfiend.lab` ↔ `apexfiend.lab` trust is summarized in `domains.md`, Section 5. The Kerberos-relevant property repeated here for context:

```text
SIDFilteringQuarantined: False
```

Because SID filtering quarantine is not applied to this intra-forest parent-child trust (expected default behavior), a Kerberos ticket issued by the child domain that carries a root-domain SID in its PAC's SID history will be accepted by the root domain. This is the mechanism underlying `attack-path.md` Stage 15 (SID History / ExtraSid abuse from `DC02$` to `DC01$`).

## 5. Validation Status

| Item | Status |
|---|---|
| Delegation configuration across users and computers | Verified (empty, as expected pre-exploitation) |
| `CA01$` RBCD attribute state | Verified (empty) |
| `m.ortega` constrained delegation state | Verified (empty) |
| Domain password and lockout policy | Verified |
| Intentionality of `LockoutThreshold: 0` | Not yet confirmed with the user |
| Kerberos ticket lifetime policy (max ticket age, renewal) | Not yet queried — requires reviewing the Default Domain Policy GPO (Account Policies → Kerberos Policy), not a `Get-ADDomain`/`Get-ADDefaultDomainPasswordPolicy` property |

## 6. Related Documentation

- `domains.md` — Forest/domain trust configuration (Section 5 covers the same trust referenced here)
- `delegated-permissions.md` — The `WriteDacl` ACE that enables the RBCD configuration described in Section 2
- `acl-design.md` — Synthesis connecting this document's findings to the full attack narrative
- `../00-overview/attack-path.md` — Stages 10–12 (RBCD) and Stage 15 (ExtraSid) both depend on findings recorded here