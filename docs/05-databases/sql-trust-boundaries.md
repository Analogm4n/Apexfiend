# Apexfiend — SQL Trust Boundaries

## 1. Purpose

This document analyzes the security boundaries crossed by the confirmed MSSQL01 → MSSQL03 → OS → j.derek attack chain, in the same spirit as `01-architecture/trust-boundaries.md` and `02-active-directory/acl-design.md`. **`MSSQL03` has replaced `MSSQL02`**; this document reflects the current configuration, including the revised SQL injection-based escalation mechanism.

## 2. The Boundaries, Individually

Taken one at a time, nothing in this chain looks dangerous:

- **A web application has its own SQL login** (`jobportal_app`), scoped with no server-level privilege.
- **A linked server lets one SQL instance query another** (`MSSQL01` → `MSSQL03`) under an explicitly mapped login, not broad remote-login access.
- **A reporting account is granted `EXECUTE` on a specific stored procedure** (`usp_GetReportAnalytics`) rather than broad database or server rights — on its face, a narrowly-scoped, reasonable grant.
- **A stored procedure runs under an elevated context** (`EXECUTE AS OWNER`) to perform a reporting task the calling account shouldn't need direct rights for — a common, legitimate pattern.

## 3. Where the Boundary Actually Breaks

The chain becomes dangerous because of the combination of two things: **`EXECUTE AS OWNER`** on the procedure, and **unparameterized dynamic SQL** built from caller-supplied input inside it. Neither alone is automatically unsafe — many `EXECUTE AS OWNER` procedures never touch dynamic SQL, and many procedures that build dynamic SQL don't run in an elevated context. Here, both are true simultaneously: `svc_reporting` only needs the `EXECUTE` grant it was legitimately given, and a single unsanitized string concatenation turns that narrow grant into arbitrary SQL running as the procedure's owner.

This differs from the escalation mechanism originally documented for `MSSQL02` (`TRUSTWORTHY ON` plus an unconditional `EXECUTE AS sql_dba` call, with no injection required) — that version needed no injection at all; simply calling the procedure was sufficient. The `MSSQL03` version is arguably the more realistic finding: SQL injection inside a privileged stored procedure is a well-documented, commonly-encountered real-world vulnerability class, whereas an unconditional elevation with no attacker input involved is less often seen outside of deliberately-vulnerable labs.

## 4. Why This Matters More Than a Single Compromised Account

As with the original chain, no password, hash, or Kerberos ticket needs to be stolen at any step. An attacker who can execute queries as `jobportal_app` — requiring only the access the `JobPortal` web application already has — can reach `svc_reporting`'s context on `MSSQL03` via the linked server, inject through `@ReportCategory`, and land in the procedure owner's elevated context entirely through legitimate SQL Server features (linked servers, `EXECUTE AS`, granted `EXECUTE` permissions) combined with one ordinary coding mistake (string concatenation instead of parameterization).

## 5. The j.derek Dimension

**New in this configuration:** `MSSQL03` also hosts `j.derek`'s "Apex Certificate Maintenance" scheduled task, which runs as `j.derek` himself every two minutes (`06-pki/certificate-inventory.md`). This means OS-level access gained through the SQL injection chain lands on the same system where `j.derek`'s own interactive-context credentials are in active, frequent use — a significantly shorter and more reliable path to his identity than would exist if the database host and the scheduled-task host were separate systems.

## 6. Status of the Linked-Server RPC Alternate Path

The previously-documented alternate path — direct `OPENQUERY`/`EXEC...AT` access via the linked server's `RPC Out`/`Data Access` settings (confirmed enabled and tested against the original `MSSQL02`) — has not been independently re-confirmed against `MSSQL03`. Given the linked server configuration was reported as carried over "exactly as in MSSQL02," this path presumably still exists, but this should be re-verified rather than assumed.

## 7. Validation Status

| Item | Status |
|---|---|
| `EXECUTE AS OWNER` + SQL injection as the root cause of the escalation | Verified (source reviewed) |
| Chain tested end-to-end, reaching OS-level access | Operator-confirmed |
| `j.derek`'s scheduled task co-located on the same host | Verified |
| Linked-server RPC/`OPENQUERY` alternate path re-confirmed on MSSQL03 | Not yet re-verified |

## 8. Related Documentation

- `mssql01.md`, `mssql03.md` — Instance-level detail
- `reportingdb.md` — The injectable procedure at the center of this analysis
- `sql-architecture.md` — Topology-level summary
- `01-architecture/trust-boundaries.md`, `02-active-directory/acl-design.md` — Parallel trust-boundary analyses elsewhere in the project
- `../06-pki/certificate-inventory.md` — The j.derek scheduled task referenced in Section 5
- `../00-overview/attack-path.md` — Stages 4–5