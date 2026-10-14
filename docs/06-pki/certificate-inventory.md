# Apexfiend — Certificate Inventory Tooling

## 1. Purpose

This document records the verified configuration of the certificate inventory tooling referenced at a design level in `00-overview/lab-setup.md` — a scheduled task running under `j.derek`'s identity that collects certificate/template data and copies it to a share on `WK03`. It is the mechanism by which `j.derek` becomes reachable from `MSSQL03` (formerly `MSSQL02` — see `05-databases/mssql03.md`) in the attack path.

## 2. Scheduled Task

```powershell
schtasks /query /tn "Apex Certificate Maintenance" /v /fo list
```

```text
Host:              MSSQL03
Task name:          Apex Certificate Maintenance
Run as user:         j.derek
Author:               CORP\admin
Task to run:          powershell.exe -NoProfile -ExecutionPolicy Bypass -File "C:\Users\j.derek\Desktop\CertificateMaintenance.ps1"
Logon mode:            Interactive/Background
Schedule:              One-time trigger (9/13/2026, 6:19 AM) + repeat every 2 minutes, no end
Stop if runs over:     72 hours
State:                 Enabled; last run succeeded (result 0)
```

**The task runs as `j.derek` himself** — not a service account acting on his behalf — meaning `j.derek`'s own credentials/session context are what execute every two minutes on `MSSQL03`. This is the live mechanism that makes compromising `MSSQL03` (via the SQL injection / `EXECUTE AS OWNER` chain documented in `05-databases/reportingdb.md`) a path to `j.derek`'s own identity, not just to a service account.

`CORP\admin`, the same lab-setup administrative account noted as "administrative noise" in `02-active-directory/users-and-groups.md`, created the task — consistent with its documented role (used to configure lab infrastructure, no narrative role of its own).

## 3. Script — CertificateMaintenance.ps1

```powershell
$Collector  = "C:\Users\j.derek\Desktop\Certificatemssql03.exe"
$ReportDir  = "C:\ProgramData\Apex\Reports"
$LogDir     = "C:\ProgramData\Apex\Logs"
$RemoteDir  = "\\wk03\PKI-Operations\Reports"
```

The script:

1. Logs its own identity (`[System.Security.Principal.WindowsIdentity]::GetCurrent()`) and hostname to `C:\ProgramData\Apex\Logs\CertificateMaintenance.log`.
2. Runs a collector binary, `Certificatemssql03.exe`, which produces CSV report files in `C:\ProgramData\Apex\Reports`. The collector's own internal behavior (what it queries — presumably AD, certificate templates, and/or SQL Server, per the design-level description in `lab-setup.md`) has not been independently analyzed in this pass.
3. Verifies the destination SMB share, `\\wk03\PKI-Operations\Reports`, is reachable.
4. Copies any CSV files not already present on the remote share — skipping duplicates rather than overwriting, and logging each action.
5. Sleeps 30 seconds as a "maintenance window" before completing.

## 4. Destination Share

```text
\\wk03\PKI-Operations\Reports
```

**Confirms `WK03` hosts a share named `PKI-Operations`**, matching the group of the same name (`02-active-directory/users-and-groups.md`) and `j.derek`'s documented role as the PKI auditor working from that workstation. The share's own permissions (who besides the task can read/write to it) have not yet been independently queried.

## 5. Role in the Attack Path

```text
MSSQL03 (ReportingDB, SQL injection in usp_GetReportAnalytics)
      │  EXECUTE AS OWNER → privileged SQL context
      ▼
xp_cmdshell or equivalent OS-level access on MSSQL03
      │
      ▼
j.derek's scheduled task runs every 2 minutes AS j.derek on this same host
      │  (session/token/credential material reachable from an OS-level foothold)
      ▼
j.derek's identity → WK03 (via \\wk03\PKI-Operations\Reports and/or
                              direct use of his credentials)
```

This is the concrete mechanism underlying `00-overview/attack-path.md` Stage 5 ("MSSQL02$ → `j.derek`" — now `MSSQL03$`, pending that document's update): OS-level access on the database host places an attacker on the same system as `j.derek`'s own recurring, interactively-authenticated scheduled task, which runs often enough (every two minutes) to be reliably caught.

## 6. Validation Status

| Item | Status |
|---|---|
| Scheduled task configuration | Verified |
| Script contents and logic | Verified |
| Destination share path and naming | Verified |
| Collector binary's internal query logic | Not independently analyzed |
| Share-level permissions on `\\wk03\PKI-Operations\Reports` | Not yet queried |

## 7. Related Documentation

- `05-databases/mssql03.md` *(pending)* — The host this task runs on
- `05-databases/reportingdb.md` *(pending update)* — The SQL-injection chain reaching OS-level access on this host
- `02-active-directory/users-and-groups.md` — `j.derek`, `CORP\admin`, and `PKI-Operations`
- `../00-overview/attack-path.md` — Stage 5