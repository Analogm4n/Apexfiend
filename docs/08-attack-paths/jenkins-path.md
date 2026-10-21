# Apexfiend — Jenkins Path (WK04 → JENKINS01 → svc_automation$)

## 1. Purpose

This document continues from `pki-path.md`: command execution as `corp\m.ortega` on `WK04`. It narrates the confirmed, tested chain from there through to control of the `svc_automation$` gMSA on `JENKINS01` — and **corrects** the credential-access technique previously assumed in `07-jenkins/gmsa.md`.

## 2. WK04 — Confirming JENKINS01 and Reaching a Low-Privilege Foothold

`m.ortega`'s own workspace on `WK04` contains local clones of two previously-undocumented Gitea repositories now resolved: `infrastructure-automation` (his own) and `jenkins-operations` (`m.alvarez`'s, cloned locally by `m.ortega`). Both document Jenkins architecture, jobs, and runbooks, confirm `jenkins01.corp.apexfiend.lab` as the controller host, reference `maintenance.ps1` as AppLocker-protected, and include `m.ortega`'s own Jenkins web-UI credentials (separate from his domain password).

```text
ping jenkins01 → 172.16.50.102 (reachable)
Test-NetConnection jenkins01 -port 3389 → TcpTestSucceeded: True
```

RDP into `JENKINS01` as `corp\m.ortega` confirms he holds **no local administrative rights** there (`whoami /priv` shows only `SeChangeNotifyPrivilege`, enabled). The Jenkins web UI, reached with the credentials found in the repos, exposes only the jobs already documented in `07-jenkins/jobs.md` — per-host health checks plus `Infrastructure-Maintenance` — and direct filesystem access to `maintenance.ps1` is denied both by ACL and (implicitly) by AppLocker.

## 3. Pivoting Through Help Desk — Recovering Laura's Real Gitea Password

Mattermost references a historical server-install documentation repository, mentioned by **Ana Torres**, described as Help Desk Lead (likely the human behind `GG-HelpDesk-Lead`, membership not previously queried). Reaching it requires Help Desk access:

- **`m.rodriguez` (Miguel)** is a dead end here — his workstation is unknown, and his Gitea credentials cannot be guessed (his domain password was changed by the operator during `helpdesk-escalation.md` and is not reused elsewhere).
- **`l.sanchez` (Laura)**, per `#it-helpdesk`, is associated with `WK05`. Her domain password does not work against Gitea (confirming, again, that Gitea authentication is independent of AD — `gitea.md`, Section 4). RDP is disabled on `WK05`, but **WinRM is enabled**, and `Enter-PSSession` succeeds with her domain credentials.

Nothing of direct interest is found in Laura's live files — but her **Recycle Bin** contains a recently-deleted `TO-DO.txt`, recovered by copying the underlying `$I`/`$R` pair out before permanent deletion. It records, in her own words, a Gitea password she had just changed:

```text
#Change gitea password ✓
l.sanchez:<password>!
```

**This Gitea credential works.** A deleted personal to-do file, not yet purged from the Recycle Bin, is the actual source of the credential — a realistic failure mode (writing a password down "temporarily," then deleting the note without emptying the bin).

## 4. The Historical Password Pattern

Inside the recovered Gitea repository (`IT-Help-Desk`, organized into `inventory`, `migrations`, `operations`, `provisioning`, `tickets`), `operations/initial-credentials.md` documents a **deprecated** initial-password format:

```text
Apex-<ROLE>-<YEAR>!
```

The document notes this pattern is deprecated but **some older, critical systems have not yet been migrated off it**. `JENKINS01` — an older host in the lab's timeline — is a natural candidate. The pattern is tried against its local `Administrator` account and **succeeds**, granting full local administrative access:

```text
whoami: jenkins01\administrator
```

## 5. AppLocker Enumeration and a Working Bypass Path

Even as local `Administrator`, the environment is confirmed to be running AppLocker in enforcing mode (`Get-Service AppIDSvc` → Running; `SrpV2` policy `AllowWindows: 0` for Exe/Dll/Script/Msi/Appx — default-deny except explicit rules). The explicit rule set includes an allow rule scoped to `C:\ProgramData\ApexAutomation\*` and several deny rules for common temp/spool/task directories, alongside the standard default allow rules for `%PROGRAMFILES%`, `%WINDIR%`, and files run by `BUILTIN\Administrators`.

**A working, unenumerated path is found and used: `C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys\`** — a legitimate, often-overlooked Windows directory not covered by any explicit rule in this policy, and writable. A copy of `cmd.exe` placed there executes successfully, confirming the bypass before it's used for the next step.

## 6. Service Binary Hijack — Becoming svc_automation$ (Corrects gmsa.md)

Querying both automation-related services confirms `07-jenkins/gmsa.md`'s findings exactly:

```text
sc.exe qc jenkins
  SERVICE_START_NAME : LocalSystem

sc.exe qc ApexAutomationAgent
  BINARY_PATH_NAME   : C:\ProgramData\ApexAutomation\Agent\ApexAutomationAgent.exe
  SERVICE_START_NAME : CORP\svc_automation$
```

**The actual technique used is a service binary hijack, not LSASS credential dumping as earlier project documentation assumed.** As local `Administrator`, a payload executable is staged in the AppLocker-bypass directory identified in Section 5, and the `ApexAutomationAgent` service — which already runs as `svc_automation$` — is reconfigured to point at it:

```powershell
Stop-Service ApexAutomationAgent
sc.exe config ApexAutomationAgent binPath= "C:\Windows\System32\Microsoft\Crypto\RSA\MachineKeys\ApexAutomationAgent.exe"
Start-Service ApexAutomationAgent
```

When the service starts, Windows's Service Control Manager launches the new binary under the configured identity — `svc_automation$` — without any credential ever needing to be extracted from memory. A reverse shell connects back, confirming control of the gMSA's own execution context.

**This corrects `07-jenkins/gmsa.md`, Section 3**, which described the credential-access step as dumping Kerberos material from LSASS on either the `ApexAutomationAgent` or Dispatcher process. The actual, tested technique is simpler and requires no in-memory credential extraction at all — local administrative rights on the host plus a writable, AppLocker-exempt path are sufficient on their own, because the gMSA's password is managed transparently by Windows and never needs to be seen directly.

## 7. Chain Summary

```text
WK04 — corp\m.ortega (from pki-path.md)
      │  Gitea repos confirm JENKINS01; low-priv RDP foothold established
      ▼
Help Desk pivot: m.rodriguez dead-ends; l.sanchez's Recycle Bin
      │  reveals her real Gitea password
      ▼
IT-Help-Desk Gitea repo → deprecated initial-password pattern
      │  still valid on JENKINS01's local Administrator
      ▼
jenkins01\administrator
      │  AppLocker enumerated; MachineKeys bypass path confirmed working
      ▼
ApexAutomationAgent service binary hijacked
      │  service restarted under its existing svc_automation$ identity
      ▼
svc_automation$ — shell received
```

## 8. Validation Status

| Item | Status |
|---|---|
| Gitea repos resolving `infrastructure-automation`/`jenkins-operations` | Tested and confirmed |
| `m.ortega`'s low-privilege foothold on JENKINS01 | Tested and confirmed |
| Laura's Gitea password recovered from a deleted Recycle Bin file | Tested and confirmed |
| Deprecated password-pattern repo and its success against JENKINS01's local Administrator | Tested and confirmed |
| AppLocker policy enumeration and the `MachineKeys` bypass path | Tested and confirmed |
| Service binary hijack of `ApexAutomationAgent` as the credential-access technique | Tested and confirmed — **corrects** the LSASS-dump assumption in `gmsa.md` |

## 9. Related Documentation

- `pki-path.md` — Precedes this document; ends with `m.ortega`/`WK04`
- `rbcd-path.md` *(pending)* — Continues from `svc_automation$` into the `WriteDacl`/RBCD stage against `CA02$`
- `../07-jenkins/gmsa.md` — Updated per Section 6 of this document
- `../07-jenkins/jobs.md`, `build-agent.md`, `automation-worker.md` — Confirmed unchanged against live configuration