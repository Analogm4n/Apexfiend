# Apexfiend — Jenkins Architecture

## 1. Purpose

This document provides a consolidated architectural overview of the automation infrastructure on `JENKINS01`, synthesizing `jobs.md`, `build-agent.md`, `gmsa.md`, and `automation-worker.md`. The real architecture is **three distinct components**, not the two ("Jenkins" and a single "build agent") implied by earlier design-level documentation.

```text
Hostname:     JENKINS01
IP address:   172.16.50.102   (VLAN 50 — PKI/Automation, shares the segment with CA02)
Domain:       corp.apexfiend.lab (domain-joined)
DNS servers:  172.16.20.100, 8.8.8.8
```

## 2. The Three Components

| Component | What it is | Runs as |
|---|---|---|
| **Jenkins** (controller) | The Jenkins service itself — web UI, job definitions, build orchestration | `LocalSystem` |
| **ApexAutomationAgent** ("Automation-Worker") | The actual Jenkins build agent/executor — connects to the controller via WebSocket and runs job steps | `svc_automation$` (gMSA) |
| **Dispatcher** | A separate PowerShell polling script — **this is the component previously referred to in project documentation as the "build agent"** — watches Jenkins job results via its REST API and triggers local maintenance actions outside of Jenkins' own execution context | `svc_automation$` (gMSA), launched as a scheduled task |

**Only the Jenkins controller service itself runs as `LocalSystem`.** Every other automation component — the build agent/worker and the Dispatcher — runs under the `svc_automation$` gMSA. This is a more significant privilege surface than earlier documentation assumed: compromising either the Jenkins build agent or the Dispatcher directly yields the gMSA's context, without needing to touch the Jenkins controller process itself.

## 3. Directory Layout

```text
C:\ProgramData\ApexAutomation\
├── Agent\              — ApexAutomationAgent (Jenkins build agent/worker)
├── Dispatcher\          — Dispatcher.ps1, its config/state/token, and Actions\
│   └── Actions\          — Maintenance.ps1 (AppLocker-protected), Certificates.ps1, SQL.ps1, Backup.ps1
├── Jenkins\              — start-agent.cmd, remoting cache, job workspaces
├── Logs\
├── Reports\              — aggregate report outputs
└── Scripts\              — secondary scripts (agent.jar, identity-test.ps1, maintenance.ps1, worker.ps1) — not AppLocker-protected, not yet analyzed in this pass; distinct from Dispatcher\Actions\Maintenance.ps1
```

**A previous administrative step used `PsExec64.exe`** (found in the `Dispatcher\` folder) to configure a service under `NT AUTHORITY\SYSTEM`. This was a one-time setup action by the operator, not an ongoing part of the lab's live configuration, and the binary has since been removed.

## 4. Summary Flow

```text
Jenkins (controller, LocalSystem)
      │  job triggers, build execution
      ▼
ApexAutomationAgent (svc_automation$)  — runs the actual job steps
      │
      ▼  (separately, polling Jenkins' REST API every 60s)
Dispatcher (svc_automation$, scheduled task)
      │  on a new successful build of an approved job
      ▼
Local Action script (Dispatcher\Actions\*.ps1) — runs under svc_automation$
```

## 5. Validation Status

| Item | Status |
|---|---|
| Host identity, domain membership | Verified |
| Three-component architecture (controller / agent / dispatcher) | Verified |
| Execution identities for all three | Verified |
| `PsExec64.exe` historical use and removal | Verified (operator-confirmed) |
| `Scripts\` directory contents and purpose | Not yet analyzed |

## 6. Related Documentation

- `jobs.md` — Jenkins job definitions and the workspace structure
- `build-agent.md` — The Dispatcher component in detail
- `gmsa.md` — `svc_automation$` configuration
- `automation-worker.md` — The ApexAutomationAgent build agent in detail
- `../00-overview/attack-path.md` — Stages 8–11