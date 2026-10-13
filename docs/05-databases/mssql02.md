# Apexfiend — MSSQL02

## 1. Purpose

This document records the verified configuration of `MSSQL02`, the reporting-database instance that is the target of the MSSQL01 → MSSQL02 linked-server pivot and the host of a confirmed server-level privilege escalation chain.

```text
Hostname:     MSSQL02
FQDN:         MSSQL02.corp.apexfiend.lab
IP address:   172.16.40.105   (VLAN 40 — Corporate Applications and Data)
Domain:       corp.apexfiend.lab (domain-joined)
DNS servers:  172.16.20.101, 172.16.20.100   (DC03, DC02)
Edition:      Microsoft SQL Server 2022 (RTM) — 16.0.1000.6 — Developer Edition
OS:           Windows Server 2022 Standard Evaluation
Instance:     MSSQLSERVER (default instance), service running
Service account: corp\svc_mssql02 (domain account — confirmed via `whoami /all`, Section 4)
```

## 2. Server Principals

```sql
SELECT name, type_desc, is_disabled, create_date FROM sys.server_principals WHERE type IN ('S','U','G') AND name NOT LIKE '##%';
```

| Login | Type | Disabled | Created |
|---|---|---|---|
| `sa` | SQL login | **Disabled** (hardening fix applied by operator — originally enabled at time of first capture) | 2003-04-08 |
| `CORP\Domain Admins` | Windows group | No | 2026-09-11 |
| `NT SERVICE\SQLWriter` | Windows login | No | 2026-09-11 |
| `NT SERVICE\Winmgmt` | Windows login | No | 2026-09-11 |
| `NT SERVICE\MSSQLSERVER` | Windows login | No | 2026-09-11 |
| `NT AUTHORITY\SYSTEM` | Windows login | No | 2026-09-11 |
| `NT SERVICE\SQLSERVERAGENT` | Windows login | No | 2026-09-11 |
| `NT SERVICE\SQLTELEMETRY` | Windows login | No | 2026-09-11 |
| `svc_reporting` | SQL login | No | 2026-09-12 |
| `sql_dba` | SQL login | No | 2026-09-24 |

**`sa` was found enabled on initial review and has since been disabled by the operator** as a hardening fix — noted here for traceability rather than left as an open finding.

## 3. sysadmin Role Membership

```text
sa
CORP\Domain Admins
NT SERVICE\SQLWriter
NT SERVICE\Winmgmt
NT SERVICE\MSSQLSERVER
NT SERVICE\SQLSERVERAGENT
sql_dba
```

**Confirmed: `sql_dba` is `sysadmin` on both `MSSQL01` and `MSSQL02`.** `svc_reporting` is **not** a member of `sysadmin` on this instance — its only confirmed server-level permission is `CONNECT SQL` (verified via `sys.server_permissions`). The escalation path from `svc_reporting` to server-level control does not rely on direct role membership; see Section 5.

## 4. Confirmed Privilege Escalation: svc_reporting → sql_dba → OS

```text
1. svc_reporting  — CONNECT SQL only; db_owner of ReportingDB
2. ReportingDB has TRUSTWORTHY ON
3. A stored procedure in ReportingDB is configured EXECUTE AS sql_dba
4. svc_reporting, as db_owner, can execute that procedure — running in the
   sql_dba security context, which is sysadmin
5. From that sysadmin context, xp_cmdshell was temporarily re-enabled and tested
```

```sql
EXEC sp_configure 'show advanced options', 1; RECONFIGURE;
EXEC sp_configure 'xp_cmdshell', 1; RECONFIGURE;
EXEC xp_cmdshell 'whoami /all';
```

**Result:** confirmed the SQL Server service runs as the domain account **`corp\svc_mssql02`** — a service account distinct from the database logins already documented, representing the actual OS-level identity reachable from this chain.

**xp_cmdshell was re-disabled immediately after the test** (`EXEC sp_configure 'xp_cmdshell', 0; RECONFIGURE;`), consistent with the lab's documented emphasis on OPSEC-aware tradecraft elsewhere (`02-active-directory/acl-design.md`, Section 3.3) — this is worth carrying into `08-attack-paths/` as a modeled "clean up after testing" step rather than leaving `xp_cmdshell` enabled.

This chain is a textbook SQL Server privilege-escalation pattern: ownership chaining enabled by `TRUSTWORTHY ON` plus an `EXECUTE AS`-configured module lets a `db_owner` with no server-level rights reach `sysadmin` without ever needing the privileged login's own credentials.

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
| `ReportingDB` | Online | Full |

See `reportingdb.md` for database-level detail (ownership, `TRUSTWORTHY` setting, the escalation procedure).

## 6. Inbound Linked Server (from MSSQL01)

No outbound linked server exists from `MSSQL02` itself (`sys.servers WHERE is_linked = 1` returned empty on this instance). `MSSQL02` is reached **inbound**, as the remote end of the linked server configured on `MSSQL01` — see `mssql01.md`, Section 4, for the local-to-remote login mapping (`jobportal_app` → `svc_reporting`).

## 7. Validation Status

| Item | Status |
|---|---|
| Host identity, domain membership, DNS | Verified |
| SQL Server version/edition | Verified |
| Server principals | Verified |
| `sysadmin` role membership (`sql_dba` confirmed on both instances) | Verified |
| `svc_reporting` → `sql_dba` → `sysadmin` escalation chain | Verified and tested end-to-end |
| Service account identity (`corp\svc_mssql02`) | Verified via `xp_cmdshell whoami /all` |
| `sa` hardening (disabled) | Applied by operator |
| Linked-server (`OPENQUERY`/`EXEC...AT`) pivot from `MSSQL01` | Status to confirm — see `sql-trust-boundaries.md` |

## 8. Related Documentation

- `mssql01.md` — The linked server and login mapping reaching this instance as `svc_reporting`
- `reportingdb.md` — Database-level detail for `ReportingDB`
- `sql-trust-boundaries.md` — Full synthesis of the MSSQL01 → MSSQL02 → OS trust chain