# Apexfiend — gMSA (svc_automation$)

## 1. Purpose

This document records the verified configuration of the `svc_automation$` Group Managed Service Account, consolidating what's confirmed here with the Active Directory-side detail already in `02-active-directory/users-and-groups.md` and `delegated-permissions.md`.

## 2. AD Configuration

```powershell
Get-ADServiceAccount svc_automation -Properties * | Select-Object PrincipalsAllowedToRetrieveManagedPassword, PrincipalsAllowedToDelegateToAccount
```

```text
PrincipalsAllowedToRetrieveManagedPassword:  GG-Jenkins-Hosts  (OU=Groups,DC=corp,DC=apexfiend,DC=lab)
PrincipalsAllowedToDelegateToAccount:          (empty)
```

Matches what was already confirmed in `02-active-directory/users-and-groups.md`, Section 6: only computer accounts in `GG-Jenkins-Hosts` (`JENKINS$`, `JENKINS01$`) are authorized to retrieve this gMSA's managed password. No constrained delegation is configured for this account (consistent with `02-active-directory/kerberos.md`, which found no delegation configured anywhere outside the domain controllers at the time of that review).

## 3. Where This Identity Actually Runs

**Confirmed usage on `JENKINS01`, revising what earlier documentation assumed:**

| Process | Runs as |
|---|---|
| Jenkins controller service | `LocalSystem` — **not** `svc_automation$` |
| `ApexAutomationAgent` (the real build agent/worker, node `Automation-Worker`) | `svc_automation$` |
| Dispatcher (scheduled task, the "build agent" of earlier documentation) | `svc_automation$` |
| Dispatcher's local action scripts (`Actions\*.ps1`, including the AppLocker-protected `Maintenance.ps1`) | `svc_automation$` (inherits the Dispatcher's own context) |

**Superseded — the real, tested technique does not involve LSASS dumping at all.** Earlier project documentation (`00-overview/lab-setup.md`, `attack-path.md`) described the gMSA's Kerberos material as obtained by dumping LSASS from "the Jenkins process" on `JENKINS01`. `08-attack-paths/jenkins-path.md` documents what actually happens once local `Administrator` access is obtained on `JENKINS01`: the `ApexAutomationAgent` service — already configured to run as `svc_automation$` (confirmed via `sc.exe qc`) — has its binary path reconfigured to point at an attacker-controlled executable (staged through a confirmed AppLocker-bypass path), and the service is restarted. The Service Control Manager launches the new binary directly under the gMSA's identity; because gMSA passwords are managed and resolved transparently by Windows, no credential material ever needs to be extracted from memory. This is a cleaner, more realistic technique than LSASS dumping and is the one actually exercised end-to-end in this lab.

## 4. Attack-Path Relevance

```text
JENKINS01 host access (via m.ortega's certificate, attack-path.md Stage 8)
      │
      ▼
ApexAutomationAgent or Dispatcher process (both run as svc_automation$)
      │  credential/Kerberos material dumping
      ▼
svc_automation$ — sole member of GG-Infrastructure-Automation
      │  WriteDacl on OU=corp-infrastructure (02-active-directory/delegated-permissions.md)
      ▼
CA01$ (now CA02$) — RBCD configuration
```

This confirms and sharpens `attack-path.md` Stages 9–11: the credential-dumping target is one of two specific, continuously-running processes (not "the Jenkins process" generically), and everything downstream (the `GG-Infrastructure-Automation` membership, the `WriteDacl` on `corp-infrastructure`) was already independently verified in `02-active-directory/`.

## 5. Validation Status

| Item | Status |
|---|---|
| gMSA password-retrieval principals | Verified (consistent with prior `02-active-directory/` findings) |
| No delegation configured | Verified (consistent with prior findings) |
| Execution identity across all JENKINS01 automation processes | Verified (operator-confirmed) |
| Credential-access technique: service binary hijack (not LSASS dump) | Tested and confirmed — see `08-attack-paths/jenkins-path.md`, Section 6 |
| Correction needed to `attack-path.md`/`lab-setup.md`'s "LSASS dump of the Jenkins process" wording | Identified; update still pending in those two documents |

## 6. Related Documentation

- `02-active-directory/users-and-groups.md`, `delegated-permissions.md` — AD-side configuration for this account
- `automation-worker.md` — The build agent process running as this identity
- `build-agent.md` — The Dispatcher process also running as this identity
- `../00-overview/attack-path.md` — Stages 9–11; wording update needed per Section 3 above