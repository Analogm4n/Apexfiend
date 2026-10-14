# Apexfiend — ReportingDB (MSSQL03)

## 1. Purpose

This document records the verified database-level detail for `ReportingDB`, now hosted on **`MSSQL03`** (which replaces `MSSQL02` — see `mssql03.md`). The privilege-escalation mechanism has changed from a direct `EXECUTE AS sql_dba` call to a SQL injection vulnerability; this document reflects the current, live configuration.

```text
Server:     MSSQL03.corp.apexfiend.lab
Database:   ReportingDB
State:      Online (presumed consistent with the original MSSQL02 configuration)
Recovery:   Full (presumed consistent with the original MSSQL02 configuration)
```

## 2. Ownership and Trust Configuration

```text
svc_reporting: db_owner of ReportingDB (presumed carried over from the MSSQL02 configuration — not independently re-queried on MSSQL03)
```

The specific mechanism connecting `svc_reporting`'s database-level access to a privileged execution context is no longer a bare `TRUSTWORTHY ON` + `EXECUTE AS sql_dba` combination — see Section 3.

## 3. The Escalation Procedure: SQL Injection in usp_GetReportAnalytics

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

**`svc_reporting` is granted `EXECUTE` on this procedure.** The procedure concatenates its `@ReportCategory` input directly into a dynamic SQL string with no parameterization, then runs it via `EXEC(@Query)` — a classic SQL injection vulnerability. Because the procedure is defined `WITH EXECUTE AS OWNER`, any query that runs inside it — including an attacker's injected payload — executes in the **owner's** security context, not the caller's. This is the same category of ownership-context escalation documented for the original `MSSQL02` configuration, but the entry point is now an injectable parameter rather than an unconditional context switch.

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

Populated with fictional sample report rows (e.g., `Monthly_Sales_Q1`, `Quarterly_Tax_Report`) solely to make the injectable query return realistic-looking results.

## 4. Role in the Attack Path

```text
MSSQL01 (jobportal_app)
      │  linked server login mapping (now targets MSSQL03)
      ▼
MSSQL03 (svc_reporting)
      │  EXECUTE granted on usp_GetReportAnalytics
      ▼
SQL injection via @ReportCategory
      │  EXECUTE AS OWNER context reached
      ▼
Privileged SQL context (owner-equivalent)
      │
      ▼
OS-level access (confirmed working, consistent with the original MSSQL02 chain)
```

The operator has confirmed this reconfigured chain reaches the same end state as the original `MSSQL02` chain — OS-level access from `svc_reporting`'s starting position, now via SQL injection rather than a direct unconditional `EXECUTE AS` call.

## 5. Validation Status

| Item | Status |
|---|---|
| Database name, hosted on MSSQL03 | Verified |
| `svc_reporting` `EXECUTE` grant on `usp_GetReportAnalytics` | Verified (source reviewed) |
| SQL injection vulnerability in `@ReportCategory` | Verified (source reviewed) |
| `EXECUTE AS OWNER` elevation | Verified (source reviewed) |
| `svc_reporting` as `db_owner` | Presumed carried over from MSSQL02; not independently re-queried |
| Full chain reaching OS-level access | Operator-confirmed as working identically to the prior chain |

## 6. Related Documentation

- `mssql03.md` — Server-level detail for the instance now hosting this database
- `mssql01.md` — The linked-server origin of this chain, repointed to MSSQL03
- `sql-trust-boundaries.md` — Full synthesis across both instances