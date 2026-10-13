# Apexfiend — SQL Architecture

## 1. Purpose

This document provides a consolidated overview of the lab's SQL Server infrastructure, summarizing `mssql01.md` and `mssql02.md` into a single topology-level reference.

## 2. Instances

| Instance | FQDN | IP | Edition | Role |
|---|---|---|---|---|
| `MSSQL01` | `MSSQL01.corp.apexfiend.lab` | `172.16.40.104` | SQL Server 2022 Developer | Hosts `JobPortal`, the database backing `WEB02`'s recruitment application |
| `MSSQL02` | `MSSQL02.corp.apexfiend.lab` | `172.16.40.105` | SQL Server 2022 Developer | Hosts `ReportingDB`; target of a confirmed privilege-escalation chain to OS-level access |

Both instances run the default `MSSQLSERVER` instance on Windows Server 2022 Standard Evaluation, are domain-joined to `corp.apexfiend.lab`, and share VLAN 40 with the rest of the corporate application/data tier.

## 3. Cross-Instance Relationship

```text
MSSQL01                                    MSSQL02
--------                                    --------
jobportal_app (SQL login)  ───linked server──▶  mapped to: svc_reporting
   not sysadmin                                    not sysadmin; db_owner on ReportingDB
   DB: JobPortal                                   DB: ReportingDB (TRUSTWORTHY ON)
                                                        │
                                                        ▼ EXECUTE AS sql_dba (via stored proc)
                                                     sql_dba = sysadmin on both instances
                                                        │
                                                        ▼ xp_cmdshell (tested)
                                                     OS access as corp\svc_mssql02
```

The single linked server, configured on `MSSQL01` pointing at `MSSQL02`, is the only cross-instance trust relationship — there is no corresponding linked server configured in the reverse direction.

## 4. Shared Principals

`sql_dba` is a member of `sysadmin` on **both** instances — the same SQL login, independently granted top-level privilege on each server. `CORP\Domain Admins` is likewise `sysadmin` on both, as a standard (if broad) default grant present on each instance independently rather than through any SQL-level cross-instance trust.

## 5. Summary of the Confirmed Attack Path

The complete, tested chain from the public-facing application down to OS-level access:

```text
WEB02 (JobPortal app)
      │  SQL Authentication
      ▼
MSSQL01 — jobportal_app
      │  linked server login mapping
      ▼
MSSQL02 — svc_reporting
      │  db_owner on ReportingDB (TRUSTWORTHY ON) + EXECUTE AS sql_dba
      ▼
sql_dba context — sysadmin
      │  xp_cmdshell
      ▼
corp\svc_mssql02 — OS-level command execution
```

This chain requires no credentials beyond what `jobportal_app` already has — every subsequent step is reached through configuration (linked-server mapping, database ownership chaining) rather than credential theft.

## 6. Validation Status

| Item | Status |
|---|---|
| Both instances' identity, version, and domain membership | Verified |
| Cross-instance linked server and login mapping | Verified |
| Full escalation chain, MSSQL01 → MSSQL02 → OS | Verified and tested end-to-end |
| `sql_dba` shared `sysadmin` status on both instances | Verified |

## 7. Related Documentation

- `mssql01.md`, `mssql02.md` — Per-instance detail
- `jobportal.md`, `reportingdb.md` — Per-database detail
- `sql-trust-boundaries.md` — Security-boundary analysis of this chain
- `../00-overview/attack-path.md` — Stage 4 (SQL Infrastructure) and Stage 5 (SQL → identity)