# Apexfiend — Jenkins Jobs

## 1. Purpose

This document records the verified Jenkins job structure on `JENKINS01`, within the `Automation` folder.

## 2. Top-Level Jobs

Four jobs are recognized by the Dispatcher (`build-agent.md`) as "approved" and trigger a local action when they succeed:

| Job | Dispatcher action | Script |
|---|---|---|
| `Infrastructure-Maintenance` | `Maintenance` | `Dispatcher\Actions\Maintenance.ps1` (AppLocker-protected) |
| `Certificate-Inventory` | `Certificates` | `Dispatcher\Actions\Certificates.ps1` |
| `SQL-Health-Check` | `SQL` | `Dispatcher\Actions\SQL.ps1` |
| `Backup-Verification` | `Backup` | `Dispatcher\Actions\Backup.ps1` |

A fifth job, `Web-Configuration-Backup`, exists in the workspace but is not in the Dispatcher's approved-job list — it does not trigger a local action.

## 3. Infrastructure-Maintenance — Per-Host Sub-Jobs

**Confirmed: `Infrastructure-Maintenance` triggers per-host sub-jobs**, one for each of the six servers covered by the lab's health-check design:

```text
workspace/Automation/
├── CA01/           (folder name predates the CA01→CA02 migration; host is now CA02 — see 06-pki/ca02.md)
│   └── Reports/ca01-health-check.txt
├── Gitea01/
│   └── Reports/gitea-health-check.txt
├── MAIL01/
│   └── Reports/mail01-health-check.txt
├── MSSQL01/
│   └── Reports/mssql01-health-check.txt
├── MSSQL02/        (folder name predates the MSSQL02→MSSQL03 migration; host is now MSSQL03 — see 05-databases/mssql03.md)
└── WEB03/
    └── Reports/web03-health-check.txt
```

**Workspace folder names have not been updated to reflect the CA01→CA02 and MSSQL02→MSSQL03 host migrations.** This is cosmetic (the folders still function correctly against the current hosts) but worth noting for consistency; the operator is aware and will reconcile naming across the project as needed.

`Gitea01`, `MAIL01`, and `WEB03` sub-job workspaces each contain a `known_hosts_<host>` file and, for `Gitea01`/`MAIL01`/`WEB03` specifically, a `secretFiles` directory under their `@tmp` workspace — consistent with SSH-based health checks against the Linux hosts (`WEB01`/`WEB03`'s SSH configuration already documented in `04-web-and-collaboration/`). `CA01` and `MSSQL01`/`MSSQL02` sub-jobs show no `known_hosts` file, consistent with WinRM/native Windows remoting rather than SSH against those Windows hosts.

## 4. Reports

```text
C:\ProgramData\ApexAutomation\Reports\
├── backup-verification.txt
├── certificate-inventory.txt
├── Infrastructure-Maintenance.txt
└── sql-health-check.txt
```

Aggregate, top-level report outputs — one per approved job — distinct from the per-host reports under each sub-job's own workspace (Section 3).

## 5. Validation Status

| Item | Status |
|---|---|
| Four approved top-level jobs and their mapped actions | Verified |
| `Web-Configuration-Backup` existing but not Dispatcher-approved | Verified |
| Per-host sub-job structure under `Infrastructure-Maintenance` | Verified (operator-confirmed) |
| Stale `CA01`/`MSSQL02` folder naming after host migrations | Verified — cosmetic, not corrected |
| SSH vs. WinRM health-check mechanism per host | Inferred from workspace artifacts, not independently confirmed |

## 6. Related Documentation

- `build-agent.md` — The Dispatcher component that maps these jobs to local actions
- `architecture.md` — Full directory layout
- `06-pki/ca02.md`, `05-databases/mssql03.md` — The hosts behind the stale `CA01`/`MSSQL02` folder names