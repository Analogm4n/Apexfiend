# Apexfiend — Build Agent (Dispatcher)

## 1. Purpose

This document records the verified configuration of the Dispatcher — the component previously referred to in project documentation as the "build agent," renamed for clarity but functionally unchanged. It is a standalone PowerShell polling script, separate from Jenkins' own execution context, that maps successful Jenkins job builds to local privileged actions.

```text
Location:   C:\ProgramData\ApexAutomation\Dispatcher\
Runs as:     svc_automation$ (gMSA), launched as a scheduled task
```

## 2. Configuration

```json
{
    "JenkinsUrl": "http://jenkins01.corp.apexfiend.lab:8080",
    "JenkinsFolder": "Automation",
    "PollIntervalSeconds": 60,
    "TokenFile": "C:\\ProgramData\\ApexAutomation\\Dispatcher\\jenkins.token",
    "StateFile": "C:\\ProgramData\\ApexAutomation\\Dispatcher\\state.json",
    "LogFile": "C:\\ProgramData\\ApexAutomation\\Dispatcher\\Logs\\dispatcher.log"
}
```

Jenkins' web interface is reached over plain HTTP (not HTTPS) on `jenkins01.corp.apexfiend.lab:8080` — internal-only traffic, consistent with the segmentation documented in `03-network-services/`, but still worth noting as unencrypted.

## 3. Authentication to Jenkins

The Dispatcher authenticates to the Jenkins REST API using HTTP Basic authentication, built from a fixed username and a token read from `jenkins.token`:

```powershell
$User = "jenkinsadmin"
$Pair = "$User`:$Token"
$Encoded = [Convert]::ToBase64String([System.Text.Encoding]::ASCII.GetBytes($Pair))
# Authorization: Basic <Encoded>
```

**The Jenkins API credential (`jenkinsadmin` + the contents of `jenkins.token`) is a separate credential from `svc_automation$` itself** — the Dispatcher process runs as the gMSA, but authenticates to Jenkins as a distinct Jenkins-local account. The token's value is not reproduced here.

## 4. Approved Jobs and Action Mapping

```powershell
$AllowedJobs = @{
    "Infrastructure-Maintenance" = "Maintenance"
    "Certificate-Inventory"      = "Certificates"
    "SQL-Health-Check"           = "SQL"
    "Backup-Verification"        = "Backup"
}
```

This is the authoritative, hard-coded allow-list — any Jenkins job not in this list is ignored entirely, regardless of its build result. **This is the real mechanism behind the project's repeated description of the dispatcher/build agent as mapping "approved build jobs to predefined scripts rather than providing unrestricted command execution"** (`00-overview/lab-setup.md`) — confirmed accurate against the live script.

## 5. Polling Logic

Every `PollIntervalSeconds` (60s), the Dispatcher:

1. Queries the Jenkins API for the `Automation` folder's jobs and each one's last build (`GET /job/Automation/api/json?tree=jobs[name,lastBuild[number,result,url]]`).
2. For each job present in `$AllowedJobs`, checks whether the last build's `result` is `SUCCESS` and its build number is greater than what was last processed (tracked per-job in `state.json`).
3. If so, invokes the corresponding script in `Dispatcher\Actions\`, passing the job name and build number as parameters.
4. **Only updates `state.json` if the action script completes without throwing** — a failed action is retried on the next poll cycle rather than silently marked as handled.

## 6. Action Scripts

```text
Dispatcher\Actions\
├── Maintenance.ps1    ← AppLocker-protected (file-hash rule); cannot be edited and have the edit execute
├── Certificates.ps1
├── SQL.ps1
└── Backup.ps1
```

**`Maintenance.ps1` is confirmed as the AppLocker-protected script** referenced throughout the attack-path design (`00-overview/attack-path.md`, `lab-setup.md`) — the one an attacker cannot simply edit, which drives the design's pivot toward dumping `svc_automation$`'s own Kerberos material from the running process rather than modifying its logic. The other three Action scripts' AppLocker status has not been independently confirmed.

## 7. Logging

Every invocation logs to `Dispatcher\Logs\dispatcher.log`, including the running identity at startup (`[System.Security.Principal.WindowsIdentity]::GetCurrent().Name`) — confirmed by the operator to read `svc_automation$`.

## 8. Validation Status

| Item | Status |
|---|---|
| Configuration (URL, folder, poll interval, file paths) | Verified |
| Execution identity (`svc_automation$`, scheduled task) | Verified (operator-confirmed) |
| Jenkins API authentication mechanism | Verified (source reviewed) |
| Approved-job allow-list | Verified (source reviewed), matches design documentation |
| Polling and state-tracking logic | Verified (source reviewed) |
| `Maintenance.ps1` as the AppLocker-protected script | Verified (operator-confirmed) |
| AppLocker status of `Certificates.ps1`/`SQL.ps1`/`Backup.ps1` | Not independently confirmed |

## 9. Related Documentation

- `architecture.md` — Where the Dispatcher fits among the three automation components
- `jobs.md` — The jobs this component watches
- `gmsa.md` — The `svc_automation$` identity this component runs as
- `../00-overview/attack-path.md` — Stages 9–11 (Jenkins → automation identity → WriteDACL)