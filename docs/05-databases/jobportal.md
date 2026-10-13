# Apexfiend — JobPortal (MSSQL01)

## 1. Purpose

This document records the verified database-level detail for `JobPortal`, hosted on `MSSQL01`.

**Naming note:** earlier design-level documentation (`00-overview/lab-setup.md`, `01-architecture/system-inventory.md`) referred to this database as `RecruitmentDB`; the live database is actually named `JobPortal`. This document is filed as `jobportal.md` rather than `recruitmentdb.md` to match the confirmed live name. The operator is updating the earlier-referencing documents manually.

```text
Server:     MSSQL01.corp.apexfiend.lab
Database:   JobPortal
State:      Online
Recovery:   Simple
```

## 2. Application Access

```text
Connecting application: JobPortal IIS site on WEB02 (../04-web-and-collaboration/web02.md)
Login:                   jobportal_app (SQL Authentication)
Connection encryption:   Disabled (Encrypt=False — already flagged as a finding in web02.md)
```

`jobportal_app` is **not** a member of `sysadmin` on `MSSQL01` (confirmed in `mssql01.md`, Section 3). Its database-level permissions within `JobPortal` itself (e.g., whether it is `db_owner` or has a narrower role) have not yet been queried.

## 3. Schema and Content

Table-level schema, row counts, and application data have not been captured in this pass — this document records connectivity and access-control facts only, consistent with the project's general approach of not reproducing application data.

## 4. Role in the Attack Path

`jobportal_app`'s significance is not its own privilege level, but its use as the **local login mapped across the MSSQL01 → MSSQL02 linked server** (`mssql01.md`, Section 4): any principal able to execute queries as `jobportal_app` — most directly, the JobPortal web application itself — can reach `MSSQL02` as `svc_reporting`, and from there into the confirmed privilege-escalation chain documented in `mssql02.md`, Section 4.

## 5. Validation Status

| Item | Status |
|---|---|
| Database existence, state, recovery model | Verified |
| Connecting application and login | Verified (cross-referenced from `web02.md`) |
| `jobportal_app`'s database-level role/permissions within `JobPortal` | Not yet queried |
| Schema/table structure | Not captured (out of scope unless needed for a specific attack-path step) |

## 6. Related Documentation

- `mssql01.md` — Server-level detail and the linked-server login mapping
- `../04-web-and-collaboration/web02.md` — The application connecting as `jobportal_app`
- `sql-trust-boundaries.md` — Full trust-chain synthesis