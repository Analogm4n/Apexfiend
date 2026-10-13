# Apexfiend — SQL Trust Boundaries

## 1. Purpose

This document analyzes the security boundaries crossed by the confirmed MSSQL01 → MSSQL02 → OS attack chain, in the same spirit as `01-architecture/trust-boundaries.md` and `02-active-directory/acl-design.md` — explaining why a sequence of individually-reasonable configurations becomes a privilege-escalation path when combined.

## 2. The Boundaries, Individually

Taken one at a time, nothing in this chain looks dangerous:

- **A web application has its own SQL login** (`jobportal_app`), scoped with no server-level privilege. This is correct practice — applications should not connect as `sa` or a sysadmin account.
- **A linked server lets one SQL instance query another** (`MSSQL01` → `MSSQL02`) under an explicitly mapped login, not a broad "remote logins enabled" setting (`is_remote_login_enabled = 0`, confirmed in `mssql01.md`). This, too, is normal practice for cross-instance reporting.
- **A reporting account owns its own reporting database** (`svc_reporting` as `db_owner` of `ReportingDB`). Database ownership of a dedicated database is unremarkable.
- **A stored procedure runs under an elevated context** (`EXECUTE AS sql_dba`) to perform an administrative task the calling account shouldn't need direct rights for. This is a common, legitimate pattern — it lets an application perform a privileged action without being privileged itself.

## 3. Where the Boundary Actually Breaks

The chain becomes dangerous specifically because of **`TRUSTWORTHY ON`** on `ReportingDB`. Without it, `EXECUTE AS` inside the database is contained to that database's own context — a module cannot use its impersonation to reach server-level resources it wouldn't otherwise have. With `TRUSTWORTHY ON`, that containment is removed: anything the database owner's modules can do, they can do with the full weight of whatever account the `EXECUTE AS` context names — in this case, a `sysadmin`.

This is the single configuration flag separating "a reporting account that can run one specific administrative report" from "a reporting account that is `sysadmin` in practice." Everything else in the chain (the linked-server mapping, the `db_owner` grant, the `EXECUTE AS` pattern itself) is standard and would be unremarkable without it.

## 4. Why This Matters More Than a Single Compromised Account

Because the chain runs through configuration rather than credential theft, **no password, hash, or Kerberos ticket needs to be stolen at any step**. An attacker who can execute queries as `jobportal_app` — which requires only the access the `JobPortal` web application already has — reaches `sysadmin`-equivalent control of `MSSQL02` and, from there, code execution as a domain service account, entirely through legitimate SQL Server features used as designed, just configured without the one safeguard (`TRUSTWORTHY OFF`, or avoiding `EXECUTE AS` on a `TRUSTWORTHY` database) that would have contained it.

## 5. Alternate Path: Linked-Server RPC

A second, independent path exists through the same linked server: `is_rpc_out_enabled = 1` and `is_data_access_enabled = 1` are both set on the `MSSQL01` → `MSSQL02` linked server configuration, meaning remote procedure calls and `OPENQUERY`/`EXEC ... AT` syntax are both available — not just simple distributed queries. This gives a principal on `MSSQL01` a second route toward `MSSQL02`, independent of the `EXECUTE AS`/`TRUSTWORTHY` chain documented above.

**Confirmed: this path has also been tested end-to-end**, alongside the `EXECUTE AS sql_dba` chain documented in Sections 2–4. Two independent, fully working routes from `MSSQL01` to privileged access on `MSSQL02` now exist: the ownership-chaining escalation via `TRUSTWORTHY`/`EXECUTE AS`, and direct RPC/`OPENQUERY` access through the linked server itself. Either is sufficient on its own; their coexistence means closing one (e.g., disabling `TRUSTWORTHY` on `ReportingDB`) would not fully remediate the underlying exposure without also addressing the linked server's RPC permissions.

## 6. OPSEC Observation

The operator re-disabled `xp_cmdshell` immediately after confirming the escalation (`mssql02.md`, Section 4). This mirrors the OPSEC-aware design philosophy already established in `02-active-directory/acl-design.md` (choosing the quieter of two available privilege paths) — here applied as a testing discipline rather than an in-lab attacker decision, but worth carrying forward into `08-attack-paths/sql-path.md` as a modeled step: a careful operator restores `xp_cmdshell` to its original (disabled) state after use, since leaving it enabled is a persistent, easily-detected indicator.

## 7. Validation Status

| Item | Status |
|---|---|
| `TRUSTWORTHY ON` as the root cause of the escalation | Verified |
| `EXECUTE AS sql_dba` chain, tested end-to-end | Verified |
| Linked-server `RPC Out`/`Data Access` alternate path | Verified and tested end-to-end |

## 8. Related Documentation

- `mssql01.md`, `mssql02.md` — Instance-level detail
- `reportingdb.md` — The `TRUSTWORTHY` database at the center of this analysis
- `sql-architecture.md` — Topology-level summary
- `01-architecture/trust-boundaries.md`, `02-active-directory/acl-design.md` — Parallel trust-boundary analyses elsewhere in the project
- `../00-overview/attack-path.md` — Stages 4–5