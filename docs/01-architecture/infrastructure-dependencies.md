# Apexfiend — Infrastructure Dependencies

## 1. Purpose

This document describes the main service dependencies within the Apexfiend lab. It explains how Active Directory, DNS, application services, databases, certificate services, mail, and automation fit together.

The goal is to provide an architectural reference for understanding service relationships and identifying how a weakness in one component may affect other systems.

Dependencies listed here describe the intended or known relationships between components. They should not be interpreted as proof that every service integration or end-to-end attack path has been tested.

## 2. Dependency Overview

Apexfiend contains several interconnected infrastructure layers:

1. **Network and remote access:** pfSense, VLANs, routing, and VPN access.
2. **Identity and name resolution:** Active Directory and DNS.
3. **User and application services:** Web systems, Gitea, Mattermost, and osTicket.
4. **Data services:** MSSQL01 and MSSQL02.
5. **Certificate infrastructure:** CA01 and Active Directory Certificate Services.
6. **Automation:** Jenkins, its worker configuration, and the `svc_automation$` group Managed Service Account.
7. **Supporting services:** MAIL01 and its integration with corporate directory identities.

These layers are related, but they do not all have the same trust level or administrative scope.

## 3. Network and Remote Access Dependencies

| Component | Depends on | Purpose |
|---|---|---|
| Internal network segments | pfSense and configured routes | Inter-network connectivity and traffic control |
| VPN clients | VPN configuration and permitted routes | Remote access to authorized lab resources |
| Internal clients | DNS and network reachability | Resolve hostnames and access services |
| Cross-segment applications | Applicable firewall rules and routes | Reach required services in other VLANs |

The availability of a route does not guarantee that a connection succeeds. Firewall rules, service listeners, host firewalls, and authentication controls can all affect connectivity.

## 4. Active Directory and DNS

### 4.1 Domain Structure

The Active Directory forest uses the root domain `apexfiend.lab` and the child domain `corp.apexfiend.lab`.

| Component | Address | Role |
|---|---|---|
| DC01 | `192.168.10.10` | Root-domain domain controller |
| DC02 | `172.16.20.100` | Corporate-domain domain controller, PDC emulator, and DNS |
| DC03 | `172.16.20.101` | Corporate-domain domain controller, global catalog, and DNS |

The root and corporate domains rely on the configured Active Directory topology and name-resolution settings for the operations that span their respective environments.

### 4.2 Dependency Relationships

- Domain-joined workstations depend on appropriate DNS configuration and network access to their domain services.
- Authentication and directory operations depend on reachable domain controllers and the required protocols.
- Services using LDAP depend on the configured directory endpoint, search base, bind identity, and access permissions.
- Forest-wide operations depend on the relevant domain and trust configuration.

The exact DNS forwarding, conditional forwarding, replication, and trust settings should be documented from the active configuration.

## 5. Web and Collaboration Services

### 5.1 Web Systems

| Component | Network | Domain status | Architectural role |
|---|---|---|---|
| WEB01 | VLAN 30 — DMZ | Not domain-joined | DMZ web system |
| WEB02 | VLAN 30 — DMZ | Not domain-joined | DMZ web system |
| WEB03 | VLAN 40 — Corporate applications | Corporate domain environment | Internal application host |

WEB01 and WEB02 are separate from the corporate domain environment. WEB03 resides in the corporate applications network and hosts internal collaboration and support services.

### 5.2 Gitea

Gitea is hosted at `172.16.40.100` in VLAN 40.

It provides source-code and project documentation hosting. Repositories may contain operational information about applications, infrastructure, and development workflows.

Its dependencies include network access to the service and the authentication and authorization mechanisms configured for repository access. Any integration with Active Directory or other identity providers should be documented only after its configuration has been verified.

### 5.3 Mattermost and osTicket

Mattermost and osTicket are hosted on WEB03 (`172.16.40.102`).

- **Mattermost** provides team communication and can contain operational context shared between departments.
- **osTicket** provides a support-ticket workflow and can contain information associated with help-desk activity.

Their availability depends on WEB03 and the services supporting each application. Any mail notifications, directory authentication, database connections, or other integrations should be recorded according to the deployed configuration.

Information exposed through collaboration and support platforms can be relevant to an assessment, but its presence does not itself establish a technical privilege escalation path.

## 6. Database Services

Apexfiend includes two SQL Server 2022 Developer Edition instances:

| Component | Address | Known role |
|---|---|---|
| MSSQL01 | Not recorded | Hosts RecruitmentDB |
| MSSQL02 | Not recorded | Hosts ReportingDB |

The exact IP addresses and database connectivity rules remain to be recorded.

### 6.1 Application and Database Relationships

Application components may depend on SQL Server for persistent data and reporting operations. These relationships should be mapped to the specific application, database, login, and permission involved.

The following items are relevant to the lab's database architecture:

- RecruitmentDB on MSSQL01.
- ReportingDB on MSSQL02.
- SQL logins and database users.
- Stored procedures and their execution permissions.
- Ownership and database configuration settings.
- Service accounts and any Windows-integrated authentication.
- Connections between database services and other application components.

Database ownership, execution permissions, impersonation, or trust-related configuration can affect the security boundary between a database principal and the SQL Server service context. The actual impact depends on the configuration and the privileges involved.

### 6.2 Dependency Validation

For each application-to-database relationship, record:

- Source application or host.
- Destination SQL Server instance.
- Database and relevant objects.
- Authentication method.
- Account or principal used.
- Required permissions.
- Whether the connection has been tested.

Do not infer a dependency merely because two systems exist in the same network segment.

## 7. Certificate Services and PKI

### 7.1 CA01

CA01 resides at `172.16.50.100` in VLAN 50. Its Enterprise CA name is `corp-CA01-CA`.

The certificate authority is part of the corporate PKI environment and depends on the relevant Windows infrastructure, including directory services and the configured certificate authority settings.

The HTTP enrollment interface at `/certsrv/` has been reachable from selected internal systems during lab testing. Reachability should be distinguished from authorization to enroll in a particular certificate template.

### 7.2 PKI-Related Dependencies

- Certificate templates and their permissions are managed through the Active Directory Certificate Services configuration.
- Enrollment depends on the template's enrollment permissions, CA configuration, and the client's ability to reach the relevant service.
- Certificate retrieval and validation may depend on directory and certificate publication settings.
- Administrative changes depend on the privileges assigned to the relevant identities.

The `UserTemporary` certificate template is part of the lab's PKI scenario. Its effective security impact depends on the template settings, enrollment permissions, issuance configuration, and the privileges associated with the resulting certificate.

### 7.3 PKI Operational Roles

The lab distinguishes between operational access and administrative authority:

- `j.derek` performs PKI auditing and certificate-template inventory from WK03, with read-only access to template information and no intended enrollment or modification rights.
- `m.ortega` performs Jenkins maintenance and monitoring associated with WK04 and does not have direct administrative rights over CA01.
- `m.alvarez` is associated with Jenkins and CA01 platform administration and holds Domain Admin privileges.

These role descriptions reflect the lab's intended scenario. Effective permissions should be verified against the relevant group memberships, ACLs, and service configuration.

## 8. Mail and Directory Integration

MAIL01 runs Ubuntu with Postfix and Dovecot and is configured to support internal mail identities.

The mail environment integrates with the corporate Active Directory directory service at `172.16.20.101`. The configured LDAP search base is:

```text
DC=corp,DC=apexfiend,DC=lab
```

The mail authentication configuration uses user principal names (UPNs). An LDAP authentication test using the configured UPN-based lookup succeeded during setup.

The mail service therefore depends on network reachability to the directory endpoint and the correctness of the LDAP bind configuration, search filter, user identity attributes, and directory permissions.

The mail host's IP address should be added to this document after it has been confirmed.

## 9. Jenkins and Automation

### 9.1 Jenkins Overview

Jenkins is used for lab automation and maintenance workflows. The configured jobs include:

- `Infrastructure-Maintenance`
- `Certificate-Inventory`
- `SQL-Health-Check`
- `Backup-Verification`

The jobs use predefined PowerShell scripts through a build agent. The configured workspace is:

```text
C:\ProgramData\ApexAutomation\Jenkins
```

The worker label is `apex-automation`.

The Jenkins server's IP address remains to be recorded.

### 9.2 Automation Dependencies

The automation environment depends on the Jenkins controller, its configured worker or agent, the job definitions, the scripts invoked by those jobs, and the permissions of the execution identity.

The `svc_automation$` group Managed Service Account (gMSA) is associated with the `GG-Jenkins-Hosts` group. The gMSA configuration was tested during setup.

The effective security impact of a Jenkins job depends on the job's permissions, the execution context, the worker configuration, script contents, and the resources accessible to the associated identity. A job's presence alone does not establish that an account can control the worker or access a particular service.

### 9.3 Cross-Service Relationships

The automation scenario connects several areas of the lab:

- Jenkins maintenance and job execution.
- PowerShell scripts and the worker execution context.
- Certificate inventory and PKI-related operations.
- SQL health checks and database connectivity.
- The `svc_automation$` identity and its configured access.

The exact credentials, permissions, and network access used by each job should be documented from the implementation rather than assumed from the job name.

## 10. Dependency and Validation Matrix

The following table distinguishes known components from dependencies that still require explicit verification.

| Relationship | Current understanding | Validation needed |
|---|---|---|
| Workstations → Active Directory | Domain operations use the configured domain services | Confirm DNS, authentication, and required ports per workstation |
| MAIL01 → corporate LDAP | UPN-based LDAP authentication was tested successfully | Record the final bind configuration and test results |
| Mattermost/osTicket → WEB03 | Both applications are hosted on WEB03 | Record database, mail, and identity integrations, if configured |
| Applications → SQL Server | RecruitmentDB and ReportingDB exist on separate instances | Map each application, account, and database connection |
| Internal clients → CA01 | Certificate enrollment interface was reachable from selected hosts | Verify enrollment rights and template-specific access |
| Jenkins → automation worker | Jobs use predefined PowerShell scripts and the `apex-automation` label | Confirm controller/agent topology and execution identity |
| Jenkins → `svc_automation$` | gMSA is associated with `GG-Jenkins-Hosts`; configuration was tested | Verify the exact execution context and effective permissions |
| DMZ → corporate services | Separate network segments define an intended boundary | Test actual allowed and denied connections |
| VPN → internal networks | VPN client network is `10.0.200.0/24` | Record routes, firewall policy, and accessible services |

## 11. Security Relevance

Infrastructure dependencies help explain how an initial foothold may lead to broader access. A weakness in an application, database permission, automation job, directory ACL, or certificate template can have consequences beyond the system where it is first discovered.

In Apexfiend, the intended assessment scenarios explore relationships across several security boundaries, including:

- Workstation access and Active Directory enumeration.
- Application and database access.
- SQL Server service contexts and domain identities.
- Certificate template configuration and enrollment permissions.
- Jenkins job execution and automation identities.
- Directory permissions and access to sensitive infrastructure.

These are areas of investigation, not a claim that every transition is exploitable or that the complete attack chain has been validated end to end.

## 12. Maintenance Notes

Update this document when a service is added, removed, reconfigured, or connected to a new identity or network segment.

For each significant dependency, record the source, destination, protocol, authentication method, account, required privileges, and validation evidence. Keep secrets, private keys, password material, tokens, and reusable credentials out of the public repository.
