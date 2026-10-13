# Apexfiend — ReportingDB (MSSQL02)

## 1. Purpose

This document records the verified database-level detail for `ReportingDB`, hosted on `MSSQL02` — the database at the center of the confirmed `svc_reporting` → `sql_dba` → `sysadmin` privilege escalation chain.

```text
Server:     MSSQL02.corp.apexfiend.lab
Database:   ReportingDB
State:      Online
Recovery:   Full
```

## 2. Ownership and Trust Configuration

```text
Owner (effective, via role):  svc_reporting — confirmed db_owner
TRUSTWORTHY:                   ON
```

**`TRUSTWORTHY ON` combined with `db_owner` membership is the configuration that makes the escalation chain documented in `mssql02.md`, Section 4 possible.** `TRUSTWORTHY` controls whether the database is permitted to access resources outside its own context using an impersonation context established inside it (such as an `EXECUTE AS` module) — with it enabled, a module inside `ReportingDB` that runs `EXECUTE AS sql_dba` can have that impersonation honored at the server level, not just within the database.

## 3. svc_reporting's Access

```sql
SELECT dp.name AS principal, r.name AS role FROM sys.database_role_members rm ...
```

```text
svc_reporting: db_owner
```

At the server level, `svc_reporting` holds only `CONNECT SQL` (`mssql02.md`, Section 3) — no server-level administrative right. `db_owner` within `ReportingDB` is the only privilege it needs, because the `TRUSTWORTHY` + `EXECUTE AS` combination does the rest.

## 4. The Escalation Procedure

A stored procedure exists within `ReportingDB` configured with `EXECUTE AS sql_dba`. `svc_reporting`, as `db_owner`, can execute it, running with `sql_dba`'s security context — which is `sysadmin` at the server level (`mssql02.md`, Section 3). The procedure's specific name and definition have not been recorded in this document; if useful for `08-attack-paths/sql-path.md` later, it should be captured there with its exact T-SQL.

## 5. Role in the Attack Path

```text
MSSQL01 (jobportal_app)
      │  linked server login mapping
      ▼
MSSQL02 (svc_reporting)
      │  db_owner on ReportingDB (TRUSTWORTHY ON)
      ▼
EXECUTE AS sql_dba  (via the stored procedure)
      │  sql_dba = sysadmin
      ▼
xp_cmdshell  (tested, then re-disabled)
      │
      ▼
OS command execution as corp\svc_mssql02
```

This is the fully confirmed and tested chain from the `JobPortal` application's own SQL login, through two SQL Server instances, to OS-level command execution under a domain service account — without ever needing credentials for `sql_dba`, `svc_mssql02`, or any Windows account directly.

## 6. Validation Status

| Item | Status |
|---|---|
| Database existence, state, recovery model | Verified |
| `TRUSTWORTHY ON` | Verified |
| `svc_reporting` as `db_owner` | Verified |
| Escalation procedure's existence and `EXECUTE AS sql_dba` configuration | Verified (exact name/definition not recorded) |
| Full chain tested end-to-end (`xp_cmdshell` → `whoami`) | Verified and tested |

## 7. Related Documentation

- `mssql02.md` — Server-level detail for the instance hosting this database
- `mssql01.md` — The linked-server origin of this chain
- `sql-trust-boundaries.md` — Full synthesis across both instances