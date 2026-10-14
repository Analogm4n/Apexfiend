# Apexfiend — MSSQL01

## 1. Purpose

This document records the verified configuration of `MSSQL01`, the SQL Server instance backing the `JobPortal` application hosted on `WEB02`.

```text
Hostname:     MSSQL01
FQDN:         MSSQL01.corp.apexfiend.lab
IP address:   172.16.40.104   (VLAN 40 — Corporate Applications and Data)
Domain:       corp.apexfiend.lab (domain-joined)
DNS servers:  172.16.20.101, 172.16.20.100   (DC03, DC02)
Edition:      Microsoft SQL Server 2022 (RTM) — 16.0.1000.6 — Developer Edition
OS:           Windows Server 2022 Standard Evaluation
Instance:     MSSQLSERVER (default instance), service running
```

## 2. Server Principals

```sql
SELECT name, type_desc, is_disabled, create_date FROM sys.server_principals WHERE type IN ('S','U','G') AND name NOT LIKE '##%';
```

| Login | Type | Disabled | Created |
|---|---|---|---|
| `sa` | SQL login | **Yes** | 2003-04-08 (default install date) |
| `CORP\Domain Admins` | Windows group | No | 2026-09-12 |
| `NT SERVICE\SQLWriter` | Windows login | No | 2026-09-12 |
| `NT SERVICE\Winmgmt` | Windows login | No | 2026-09-12 |
| `NT SERVICE\MSSQLSERVER` | Windows login | No | 2026-09-12 |
| `NT AUTHORITY\SYSTEM` | Windows login | No | 2026-09-12 |
| `NT SERVICE\SQLSERVERAGENT` | Windows login | No | 2026-09-12 |
| `NT SERVICE\SQLTELEMETRY` | Windows login | No | 2026-09-12 |
| `jobportal_app` | SQL login | No | 2026-09-12 |
| `sql_dba` | SQL login | No | 2026-09-24 |

`sa` is confirmed **disabled** on this instance.

## 3. sysadmin Role Membership

```sql
SELECT r.name AS role, m.name AS member FROM sys.server_role_members rm ...
```

```text
sa
CORP\Domain Admins
NT SERVICE\SQLWriter
NT SERVICE\Winmgmt
NT SERVICE\MSSQLSERVER
NT SERVICE\SQLSERVERAGENT
sql_dba
```

**`jobportal_app` is not a member of `sysadmin`** — the application login used by `WEB02`'s JobPortal app has no server-wide administrative rights on `MSSQL01` itself. `CORP\Domain Admins` holding `sysadmin` is a standard (if broad) default grant, consistent with `sql_dba` and the NT SERVICE accounts also present.

## 4. Linked Server — MSSQL03 (replaces MSSQL02)

```sql
SELECT name, product, provider, data_source, is_remote_login_enabled FROM sys.servers WHERE is_linked = 1;
EXEC sp_helplinkedsrvlogin;
```

```text
Linked server name:    MSSQL02   (local alias name; see note below)
Provider:               MSOLEDBSQL
Data source:            MSSQL03.corp.apexfiend.lab   (repointed — MSSQL03 replaces MSSQL02; see mssql03.md)
Remote login enabled:   0 (false — explicit login mapping only, no broad remote access)

Login mapping:
  Local login: jobportal_app  →  Remote login on MSSQL03: svc_reporting
```

**Note:** `MSSQL03` has replaced `MSSQL02` as the actual reporting-database host; the linked server's data source was repointed accordingly and confirmed working by the operator. The linked server object's local name may still read `MSSQL02` pending a cleanup rename — this document records the live `data_source`, which is authoritative.

**This confirms the precise mechanism behind the MSSQL01 → MSSQL03 stage of `00-overview/attack-path.md`:** any query executed through this linked server using the `jobportal_app` local login is automatically re-authenticated on `MSSQL03` as `svc_reporting`, via an explicit (non-self) login mapping. A principal with access to run queries as `jobportal_app` on `MSSQL01` — such as the JobPortal application itself, or anyone who compromises it — can therefore reach `MSSQL03` under the `svc_reporting` identity without needing `svc_reporting`'s own credentials.

`svc_reporting`'s privilege level on the remote end is documented in `mssql03.md` — `svc_reporting` is not `sysadmin` there either; access to a privileged context is instead reached via a SQL injection vulnerability in a stored procedure (`usp_GetReportAnalytics`, `EXECUTE AS OWNER`), confirmed working end-to-end.

## 5. Databases

```sql
SELECT name, state_desc, recovery_model_desc FROM sys.databases;
```

| Database | State | Recovery model |
|---|---|---|
| `master` | Online | Simple |
| `tempdb` | Online | Simple |
| `model` | Online | Full |
| `msdb` | Online | Simple |
| `JobPortal` | Online | Simple |

**`JobPortal` is the real name of the application database backing `WEB02`.** Earlier design-level documentation (`00-overview/lab-setup.md`, `01-architecture/system-inventory.md`) referred to this as `RecruitmentDB` — that name does not exist on the live server. The user is reconciling the naming across those documents manually; this document and `recruitmentdb.md`/`jobportal.md` (naming pending confirmation) use `JobPortal`, the confirmed live name.

## 6. Validation Status

| Item | Status |
|---|---|
| Host identity, domain membership, DNS | Verified |
| SQL Server version/edition | Verified |
| Server principals and `sa` disabled state | Verified |
| `sysadmin` role membership | Verified |
| Linked server configuration and login mapping | Verified |
| Database inventory | Verified |
| `svc_reporting`'s effective privilege on `MSSQL03` | Verified — see `mssql03.md` |

## 7. Related Documentation

- `mssql03.md` — The linked server's remote end (replaces `mssql02.md`); `svc_reporting`'s actual permissions
- `recruitmentdb.md` *(pending rename to `jobportal.md`?)* — Database-level detail for `JobPortal`
- `../04-web-and-collaboration/web02.md` — The application connecting as `jobportal_app`
- `sql-trust-boundaries.md` — Synthesis of the MSSQL01→MSSQL03 trust relationship