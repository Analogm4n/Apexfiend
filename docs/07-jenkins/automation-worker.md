# Apexfiend — Automation Worker (ApexAutomationAgent)

## 1. Purpose

This document records the verified configuration of `ApexAutomationAgent` — the real Jenkins build agent/executor (registered in Jenkins as the node `Automation-Worker`), distinct from both the Jenkins controller and the Dispatcher.

## 2. Service Configuration

```xml
<service>
  <id>ApexAutomationAgent</id>
  <name>Apex Automation Agent</name>
  <description>Jenkins automation worker for Apex infrastructure maintenance</description>
  <executable>C:\Program Files\Java\jdk-21.0.12\bin\java.exe</executable>
  <arguments>
    -jar "C:\ProgramData\ApexAutomation\Agent\agent.jar"
    -url "http://jenkins01.corp.apexfiend.lab:8080/"
    -secret "@C:\ProgramData\ApexAutomation\Agent\secret-file"
    -name "Automation-Worker"
    -webSocket
    -workDir "C:\ProgramData\ApexAutomation\Agent"
  </arguments>
  <onfailure action="restart" delay="10 sec" />
</service>
```

Installed as a Windows service via WinSW (the XML service wrapper format). Runs `agent.jar` (Jenkins' standard remoting agent) under Java 21, connecting outbound to the Jenkins controller over WebSocket — no inbound port needs to be opened on `JENKINS01` for this connection.

**Runs as `svc_automation$`** (the gMSA), confirmed by the operator — not `LocalSystem`, despite the service XML not explicitly declaring a `<serviceaccount>` block (the account is configured via the Windows Service Control Manager directly rather than in the WinSW XML itself).

## 3. Connection Secret

```text
secret-file   — the per-agent connection secret Jenkins issues to authorize this node
```

This file's contents are not reproduced here. Its presence under `C:\ProgramData\ApexAutomation\Agent\` means any principal able to read files there (i.e., `svc_automation$`, or anyone who compromises that context) can read the agent's connection secret.

## 4. Remoting and Workspace

```text
Agent\remoting\jarCache\   — cached remoting libraries (standard Jenkins agent artifacts)
Agent\remoting\logs\        — remoting.log.0 through .4
```

The agent's actual job workspaces are under `Jenkins\workspace\Automation\` (documented per-job in `jobs.md`), separate from the `Agent\` directory itself.

## 5. A Second Agent-Start Mechanism

```text
C:\ProgramData\ApexAutomation\Jenkins\start-agent.cmd
```

```batch
"C:\Program Files\Java\jdk-21.0.12\bin\java.exe" ^
 -jar agent.jar ^
 -url http://jenkins01.corp.apexfiend.lab:8080/ ^
 -secret @secret-file ^
 -name "Automation-Worker" ^
 -webSocket ^
 -workDir "C:\ProgramData\ApexAutomation\Jenkins"
```

A second, manual-launch batch script exists with near-identical parameters to the `ApexAutomationAgent` Windows service, but pointed at a different working directory (`...\Jenkins` rather than `...\Agent`) and its own separate `agent.jar`/`secret-file`/remoting cache. Its relationship to the service-based agent — whether it's a manual fallback, a leftover from initial setup, or something still actively used — has not been clarified.

## 6. Validation Status

| Item | Status |
|---|---|
| Service configuration (executable, arguments, connection target) | Verified |
| Execution identity (`svc_automation$`) | Verified (operator-confirmed) |
| Agent node name (`Automation-Worker`) | Verified |
| Purpose and status of `start-agent.cmd` relative to the service | Not yet clarified |

## 7. Related Documentation

- `architecture.md` — Where this component fits among the three automation pieces
- `build-agent.md` — The separate Dispatcher component, also running as `svc_automation$`
- `gmsa.md` — `svc_automation$` configuration