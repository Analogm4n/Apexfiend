# Apexfiend — WEB02

## 1. Purpose

This document records the verified configuration of `WEB02`, the Windows Server Core host in the DMZ that hosts the Job Recruitment application linked from `WEB01`'s public landing page.

```text
Hostname:     WEB02
OS:           Microsoft Windows Server 2022 Standard Evaluation (Build 20348)
IP address:   172.16.30.102   (VLAN 30 — DMZ)
Subnet mask:  255.255.255.0
Gateway:      172.16.30.1
Domain:       Not domain-joined
```

`WEB02` shares VLAN 30 with `WEB01` (`172.16.30.100`). `job.apexfiend.lab`, advertised from `WEB01`'s public landing page, resolves via `WEB01`'s local, isolated DNS zone (see `web01.md`, Section 5) directly to this host — `WEB01` only links out to it; the application itself is served entirely from `WEB02`.

## 2. IIS Site

```powershell
Get-Website | Select-Object Name, State, PhysicalPath, ApplicationPool
```

```text
Name:          JobPortal
State:         Started
PhysicalPath:  C:\inetpub\JobPortal
AppPool:       JobPortal
```

```powershell
Get-WebBinding | Select-Object protocol, bindingInformation
```

```text
Protocol: https
Binding:  *:443:job.apexfiend.lab
```

Only an HTTPS binding exists, scoped to the `job.apexfiend.lab` host header — there is no HTTP (port 80) site binding configured. IIS version: `10.0` (IIS 10, matching Windows Server 2022).

## 3. Application Pool Identity

```powershell
Get-ItemProperty "IIS:\AppPools\JobPortal" -Name processModel.identityType, processModel.userName
```

```text
userName: (empty)
```

The `userName` property is empty, consistent with the pool running under the default `ApplicationPoolIdentity` rather than a configured domain or local service account. `identityType` was not captured in this pass and should be confirmed directly if the distinction matters later (e.g., for a privilege-escalation path local to this host).

## 4. Database Connectivity

```powershell
Select-String -Path "C:\inetpub\JobPortal\web.config" -Pattern 'Data Source|Server=|Initial Catalog|Database='
```

```text
Data Source=mssql01.corp.apexfiend.lab,1433
Initial Catalog=JobPortal
User ID=jobportal_app
Password=<redacted>
Encrypt=False
```

The application authenticates to `MSSQL01` using **SQL Authentication** with a dedicated application login (`jobportal_app`), against the `JobPortal` database, over the standard SQL Server TCP port (1433). No database password is recorded in this document.

**Verified finding:** `Encrypt=False` — the connection between `WEB02` and `MSSQL01` is unencrypted. SQL authentication credentials and query traffic between a DMZ, non-domain-joined host and the corporate SQL Server instance cross the network in plaintext. This is a genuine finding, not an intentional attack-path design element, and should be treated the same way as the pfSense egress findings in `03-network-services/pfsense.md` — a candidate for `09-reporting/findings.md`.

**Needs verification:** `WEB02` is not domain-joined, yet it successfully resolves `mssql01.corp.apexfiend.lab` — an Active-Directory-integrated DNS name. The `ipconfig /all` capture used to document this host's networking did not include its configured DNS server(s). This should be confirmed (either a corporate DNS server is configured directly on the NIC, or a static `hosts` entry exists) before drawing conclusions about what else `WEB02` might be able to resolve or reach.

## 5. Host Firewall

```powershell
Get-NetFirewallRule -Enabled True -Direction Inbound | Select-Object DisplayName, Action | Where-Object {$_.DisplayName -notlike "Core*"}
```

Beyond the standard Windows Server default rules (Delivery Optimization, mDNS, AllJoyn Router — none of which are specific to this host's role), the relevant enabled inbound rules are:

```text
World Wide Web Services (HTTPS Traffic-In)   Allow
World Wide Web Services (HTTP Traffic-In)    Allow
World Wide Web Services (QUIC Traffic-In)    Allow
Allow HTTPS 443                               Allow  (custom rule)
```

The default IIS HTTP-Traffic-In rule is enabled at the host firewall level, but — per Section 2 — no HTTP site binding actually exists in IIS, so this is a dormant allowance rather than an active exposure.

## 6. Validation Status

| Item | Status |
|---|---|
| OS, hostname, IP configuration | Verified |
| IIS site and binding configuration | Verified |
| Application pool identity (`userName`) | Verified empty; `identityType` not captured |
| Database connection target, login, and encryption setting | Verified |
| Database password | Not recorded (redacted) |
| DNS resolution mechanism for `mssql01.corp.apexfiend.lab` | Not yet verified |
| Host firewall inbound rules | Verified |

## 7. Related Documentation

- `web01.md` — Public landing page that links to `job.apexfiend.lab`
- `03-network-services/dns.md` — The AD-integrated zone serving `mssql01.corp.apexfiend.lab`
- `03-network-services/pfsense.md` — DMZ network boundary `WEB02` shares with `WEB01`
- `05-databases/` *(pending)* — `MSSQL01`/`JobPortal` database detail