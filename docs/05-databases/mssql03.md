# Apexfiend — MSSQL03

## 1. Purpose

This document records the verified configuration of `MSSQL03`, which **replaces `MSSQL02`** as the reporting-database instance in the attack chain. The host, database, and linked-server configuration were rebuilt to match `MSSQL02` exactly; only the privilege-escalation mechanism inside `ReportingDB` was deliberately changed — from a direct `EXECUTE AS sql_dba` call to a SQL injection vulnerability. `mssql02.md` is now superseded by this document.

```text
Hostname:     MSSQL03
IP address:   172.16.40.107   (VLAN 40 — Corporate Applications and Data)
Domain:       corp.apexfiend.lab (domain-joined, presumed consistent with MSSQL02's configuration)
Edition:      Microsoft SQL Server 2022 Developer Edition (presumed, matching MSSQL02's build — not independently re-verified on this host)
```

## 2. Relationship to MSSQL01

`MSSQL01`'s linked server (`05-databases/mssql01.md`, Section 4) has been **repointed to `MSSQL03.corp.apexfiend.lab`**, confirmed working. The local-to-remote login mapping (`jobportal_app` → `svc_reporting`) is unchanged — only the target host moved. `mssql01.md` should be updated to reflect `MSSQL03` as the linked server's data source; this is noted here as the authoritative current state.

## 3. svc_reporting and the Privilege Escalation Chain

Unlike the original `MSSQL02` configuration (direct `EXECUTE AS sql_dba` inside a stored procedure), `MSSQL03`'s `ReportingDB` now contains a **SQL injection vulnerability** in a stored procedure, `usp_GetReportAnalytics`, created `WITH EXECUTE AS OWNER` and granted to `svc_reporting`:

```sql
USE ReportingDB;
GO

CREATE PROCEDURE dbo.usp_GetReportAnalytics
    @ReportCategory NVARCHAR(100)
WITH EXECUTE AS OWNER
AS
BEGIN
    DECLARE @Query NVARCHAR(MAX);
    SET @Query = N'SELECT LogID, ReportName, ExecutionStatus, ExecutionDate 
                   FROM dbo.ReportAuditLog 
                   WHERE ReportCategory = ''' + @ReportCategory + '''';
    EXEC(@Query);
END;
GO
GRANT EXECUTE ON dbo.usp_GetReportAnalytics TO svc_reporting;
GO
```

**The vulnerability:** `@ReportCategory` is concatenated directly into a dynamic SQL string with no parameterization or sanitization, then executed via `EXEC(@Query)` — classic SQL injection. Because the procedure runs `WITH EXECUTE AS OWNER`, an injected query executes with the privileges of the procedure's owner (the database owner context), not the caller's (`svc_reporting`'s) own, more limited rights. This is the same underlying class of privilege escalation documented previously for `MSSQL02` (ownership-context elevation via a `TRUSTWORTHY`/`EXECUTE AS` combination) — the difference is the **entry mechanism**: previously, `svc_reporting` only needed `EXECUTE` permission and the procedure did the escalation unconditionally; now, `svc_reporting` (or anything that can call this procedure with an attacker-controlled parameter) must successfully inject SQL through `@ReportCategory` to reach the same elevated context.

This is arguably a more realistic vulnerability pattern — dynamic SQL injection inside a privileged stored procedure is a common real-world finding, whereas an unconditional `EXECUTE AS` escalation with no injection required is less commonly seen in practice.

### Supporting table

```sql
CREATE TABLE dbo.ReportAuditLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    ReportName NVARCHAR(150),
    ExecutedBy NVARCHAR(100),
    ExecutionStatus NVARCHAR(50),
    ExecutionDate DATETIME DEFAULT GETDATE(),
    ReportCategory NVARCHAR(100)
);
```

Populated with fictional sample data (report names like `Monthly_Sales_Q1`, `Employee_Payroll_Summary`) purely to give the injectable query realistic-looking results — not narratively significant itself.

## 4. Confirmed: Chain Still Reaches OS-Level Access

The operator has confirmed this reconfigured chain **behaves identically to the original `MSSQL02` chain** in its end effect: `svc_reporting`, via the injectable `usp_GetReportAnalytics` procedure, still reaches a privileged execution context capable of OS-level command execution (consistent with the `xp_cmdshell`/`whoami` test previously performed against `MSSQL02`, documented in what is now historical detail in the superseded `mssql02.md`). The specific re-test of `xp_cmdshell`/`whoami` against `MSSQL03` itself has not been independently re-confirmed in this document — see Section 6.

## 5. Certificate Inventory Task

`MSSQL03` also hosts the "Apex Certificate Maintenance" scheduled task running as `j.derek`, which is the mechanism reaching `j.derek`'s identity from this host — see `../06-pki/certificate-inventory.md` for full detail. This task's presence on `MSSQL03` (rather than a separate host) means OS-level access gained via the SQL injection chain above lands directly on the same system `j.derek`'s own recurring task executes on.

## 6. Validation Status

| Item | Status |
|---|---|
| Host identity (hostname, IP) | Verified |
| Replaces `MSSQL02`; linked server repointed and confirmed working | Verified |
| SQL injection vulnerability in `usp_GetReportAnalytics` | Verified (source reviewed) |
| `EXECUTE AS OWNER` elevation mechanism | Verified (source reviewed) |
| End-to-end OS-level access via this chain | Operator-confirmed as working identically to the prior `MSSQL02` chain; not independently re-tested in this document |
| Domain membership, edition, other host-level detail | Presumed consistent with `MSSQL02`; not independently re-verified |

## 7. Related Documentation

- `mssql01.md` *(needs update)* — Linked server data source should be updated from `MSSQL02` to `MSSQL03`
- `reportingdb.md` *(pending update)* — Database-level detail needs updating to reflect the new injection-based mechanism
- `sql-architecture.md`, `sql-trust-boundaries.md` *(pending update)* — Both reference `MSSQL02` and need updating to `MSSQL03`
- `../06-pki/certificate-inventory.md` — The `j.derek` scheduled task hosted on this server