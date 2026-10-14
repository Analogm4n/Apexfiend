\# Apexfiend — ReportingDB (MSSQL03)



\## 1. Purpose



This document records the verified database-level detail for `ReportingDB`, now hosted on \*\*`MSSQL03`\*\* (which replaces `MSSQL02` — see `mssql03.md`). The privilege-escalation mechanism has changed from a direct `EXECUTE AS sql\_dba` call to a SQL injection vulnerability; this document reflects the current, live configuration.



```text

Server:     MSSQL03.corp.apexfiend.lab

Database:   ReportingDB

State:      Online (presumed consistent with the original MSSQL02 configuration)

Recovery:   Full (presumed consistent with the original MSSQL02 configuration)

```



\## 2. Ownership and Trust Configuration



```text

svc\_reporting: db\_owner of ReportingDB (presumed carried over from the MSSQL02 configuration — not independently re-queried on MSSQL03)

```



The specific mechanism connecting `svc\_reporting`'s database-level access to a privileged execution context is no longer a bare `TRUSTWORTHY ON` + `EXECUTE AS sql\_dba` combination — see Section 3.



\## 3. The Escalation Procedure: SQL Injection in usp\_GetReportAnalytics



```sql

USE ReportingDB;

GO



CREATE PROCEDURE dbo.usp\_GetReportAnalytics

&#x20;   @ReportCategory NVARCHAR(100)

WITH EXECUTE AS OWNER

AS

BEGIN

&#x20;   DECLARE @Query NVARCHAR(MAX);

&#x20;   SET @Query = N'SELECT LogID, ReportName, ExecutionStatus, ExecutionDate 

&#x20;                  FROM dbo.ReportAuditLog 

&#x20;                  WHERE ReportCategory = ''' + @ReportCategory + '''';

&#x20;   EXEC(@Query);

END;

GO

GRANT EXECUTE ON dbo.usp\_GetReportAnalytics TO svc\_reporting;

GO

```



\*\*`svc\_reporting` is granted `EXECUTE` on this procedure.\*\* The procedure concatenates its `@ReportCategory` input directly into a dynamic SQL string with no parameterization, then runs it via `EXEC(@Query)` — a classic SQL injection vulnerability. Because the procedure is defined `WITH EXECUTE AS OWNER`, any query that runs inside it — including an attacker's injected payload — executes in the \*\*owner's\*\* security context, not the caller's. This is the same category of ownership-context escalation documented for the original `MSSQL02` configuration, but the entry point is now an injectable parameter rather than an unconditional context switch.



\### Supporting table



```sql

CREATE TABLE dbo.ReportAuditLog (

&#x20;   LogID INT IDENTITY(1,1) PRIMARY KEY,

&#x20;   ReportName NVARCHAR(150),

&#x20;   ExecutedBy NVARCHAR(100),

&#x20;   ExecutionStatus NVARCHAR(50),

&#x20;   ExecutionDate DATETIME DEFAULT GETDATE(),

&#x20;   ReportCategory NVARCHAR(100)

);

```



Populated with fictional sample report rows (e.g., `Monthly\_Sales\_Q1`, `Quarterly\_Tax\_Report`) solely to make the injectable query return realistic-looking results.



\## 4. Role in the Attack Path



```text

MSSQL01 (jobportal\_app)

&#x20;     │  linked server login mapping (now targets MSSQL03)

&#x20;     ▼

MSSQL03 (svc\_reporting)

&#x20;     │  EXECUTE granted on usp\_GetReportAnalytics

&#x20;     ▼

SQL injection via @ReportCategory

&#x20;     │  EXECUTE AS OWNER context reached

&#x20;     ▼

Privileged SQL context (owner-equivalent)

&#x20;     │

&#x20;     ▼

OS-level access (confirmed working, consistent with the original MSSQL02 chain)

```



The operator has confirmed this reconfigured chain reaches the same end state as the original `MSSQL02` chain — OS-level access from `svc\_reporting`'s starting position, now via SQL injection rather than a direct unconditional `EXECUTE AS` call.



\## 5. Validation Status



| Item | Status |

|---|---|

| Database name, hosted on MSSQL03 | Verified |

| `svc\_reporting` `EXECUTE` grant on `usp\_GetReportAnalytics` | Verified (source reviewed) |

| SQL injection vulnerability in `@ReportCategory` | Verified (source reviewed) |

| `EXECUTE AS OWNER` elevation | Verified (source reviewed) |

| `svc\_reporting` as `db\_owner` | Presumed carried over from MSSQL02; not independently re-queried |

| Full chain reaching OS-level access | Operator-confirmed as working identically to the prior chain |



\## 6. Related Documentation



\- `mssql03.md` — Server-level detail for the instance now hosting this database

\- `mssql01.md` — The linked-server origin of this chain, repointed to MSSQL03

\- `sql-trust-boundaries.md` — Full synthesis across both instances

