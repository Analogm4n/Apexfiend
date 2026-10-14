# Apexfiend — SQL Architecture

## 1. Purpose

This document provides a consolidated overview of the lab's SQL Server infrastructure, summarizing `mssql01.md` and `mssql03.md` into a single topology-level reference. **`MSSQL03` has replaced `MSSQL02`** as the reporting-database host; this document reflects the current, live topology.

## 2. Instances

| Instance | FQDN | IP | Edition | Role |
|---|---|---|---|---|
| `MSSQL01` | `MSSQL01.corp.apexfiend.lab` | `172.16.40.104` | SQL Server 2022 Developer | Hosts `JobPortal`, the database backing `WEB02`'s recruitment application |
| `MSSQL03` | `MSSQL03.corp.apexfiend.lab` | `172.16.40.107` | SQL Server 2022 Developer (presumed, carried over from MSSQL02) | Hosts `ReportingDB`; also runs `j.derek`'s certificate-maintenance scheduled task (`06-pki/certificate-inventory.md`) |

`MSSQL02` is retired from the live lab topology; `mssql02.md` is superseded by `mssql03.md` and retained only for historical reference.

Both instances run the default `MSSQLSERVER` instance on Windows Server 2022, are domain-joined to `corp.apexfiend.lab`, and share VLAN 40 with the rest of the corporate application/data tier.

## 3. Cross-Instance Relationship

```text
MSSQL01                                    MSSQL03
--------                                    --------
jobportal_app (SQL login)  ───linked server──▶  mapped to: svc_reporting
   not sysadmin                                    not sysadmin; EXECUTE on usp_GetReportAnalytics
   DB: JobPortal                                   DB: ReportingDB
                                                        │
                                                        ▼ SQL injection via @ReportCategory
                                                     EXECUTE AS OWNER context reached
                                                        │
                                                        ▼ OS-level access (confirmed)
                                                     corp\svc_mssql02-equivalent service identity
                                                        │
                                                        ▼ same host as j.derek's scheduled task
                                                     j.derek's identity reachable
```

The single linked server, configured on `MSSQL01` and now pointing at `MSSQL03`, remains the only cross-instance trust relationship.

## 4. Shared Principals

`sql_dba` was previously confirmed as `sysadmin` on both the original `MSSQL01` and `MSSQL02` instances. Whether `sql_dba` carries the same role on `MSSQL03` has not been independently re-confirmed, since the escalation chain no longer depends on it directly (the new mechanism uses `EXECUTE AS OWNER` rather than impersonating `sql_dba` by name).

## 5. Summary of the Confirmed Attack Path

```text
WEB02 (JobPortal app)
      │  SQL Authentication
      ▼
MSSQL01 — jobportal_app
      │  linked server login mapping (→ MSSQL03)
      ▼
MSSQL03 — svc_reporting
      │  EXECUTE on usp_GetReportAnalytics → SQL injection → EXECUTE AS OWNER
      ▼
Privileged SQL context
      │  OS-level command execution (confirmed)
      ▼
Same host as j.derek's "Apex Certificate Maintenance" scheduled task
      │
      ▼
j.derek's identity reachable
```

This chain still requires no credentials beyond what `jobportal_app` already has to begin — every subsequent step is reached through configuration and an injectable parameter rather than credential theft, consistent with the project's broader design philosophy (`00-overview/objectives.md`).

## 6. Validation Status

| Item | Status |
|---|---|
| `MSSQL01` identity, version, domain membership | Verified |
| `MSSQL03` identity (hostname, IP) | Verified |
| `MSSQL03`'s other host-level detail (edition, domain membership) | Presumed consistent with MSSQL02; not independently re-verified |
| Cross-instance linked server, repointed to MSSQL03 | Verified and confirmed working |
| Full escalation chain, MSSQL01 → MSSQL03 → OS → j.derek | Operator-confirmed working end-to-end |

## 7. Related Documentation

- `mssql01.md`, `mssql03.md` — Per-instance detail
- `jobportal.md`, `reportingdb.md` — Per-database detail
- `sql-trust-boundaries.md` — Security-boundary analysis of this chain
- `../06-pki/certificate-inventory.md` — The j.derek scheduled task hosted on MSSQL03
- `../00-overview/attack-path.md` — Stage 4 (SQL Infrastructure) and Stage 5 (SQL → identity)