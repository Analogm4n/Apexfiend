# Apexfiend — System Inventory

## Overview

This document records the known systems deployed in the Apexfiend laboratory.

The inventory provides a centralized reference for hostnames, IP addresses, network roles, operating environments, and relevant services.

It is intended to support:

- Infrastructure administration
- Network and service enumeration
- Attack-path documentation
- Troubleshooting
- Configuration validation
- Documentation maintenance

The inventory reflects currently documented information. Fields that have not been confirmed are explicitly marked as unknown or not recorded rather than populated with assumed values.

---

## Inventory Conventions

The following conventions apply throughout this document.

### System status

| Status | Meaning |
|---|---|
| Implemented | The system or component has been configured in the laboratory. |
| Tested | The relevant system or functionality has been independently verified. |
| In development | Configuration or integration is still being developed. |
| Planned | The system or functionality is part of the intended design but has not been implemented. |
| Needs verification | Available documentation does not establish the current configuration or validation state. |

A system may be implemented even when a particular attack path involving it remains untested.

### Addressing

IP addresses are included where they have been recorded in the laboratory documentation.

An address marked `Not recorded` should be checked against the current VirtualBox and guest operating-system configuration.

### Operating systems

Only explicitly known operating-system information is included. Exact versions should be added after verification.

---

## 1. Network Infrastructure

| Component | Known address | Role | Status |
|---|---|---|---|
| pfSense | WAN: `10.0.100.6` | Gateway, firewall, routing, and VPN | Implemented |
| Development workstation | `10.0.100.11` | Simulated external/development access context | Implemented |
| VPN-related endpoint | `10.0.200.0/24` | VPN client addressing | Implemented |

The VPN subnet is separate from the development network. Individual VPN client addresses and interface assignments should be documented only after verification.

---

## 2. Active Directory Infrastructure

| Hostname | FQDN / Domain | IP address | Role | Status |
|---|---|---|---|---|
| DC01 | `DC01.apexfiend.lab` | `192.168.10.10` | Root-domain controller | Tested |
| DC02 | `DC02.corp.apexfiend.lab` | `172.16.20.100` | Corporate-domain controller, PDC and DNS | Tested |
| DC03 | `DC03.corp.apexfiend.lab` | `172.16.20.101` | Corporate-domain controller, Global Catalog and DNS | Tested |

The documented forest structure is:

```text
apexfiend.lab
└── corp.apexfiend.lab
```

The directory infrastructure has been checked using Active Directory diagnostic procedures, including `dcdiag`.

This entry records the two documented domains. Any additional domain should be added after its deployment and configuration have been confirmed.

---

## 3. Windows Workstations

| Hostname | IP address | Network role | Relevant context |
|---|---|---|---|
| WK01 | `172.16.60.101` | Standard corporate workstation | Internal enumeration and workstation access |
| WK02 | `172.16.60.102` | Web Developer / DevOps Engineer workstation | Development and web operations |
| WK03 | `172.16.60.103` | PKI-related workstation | Certificate inventory and template enumeration |
| WK04 | `172.16.60.104` | Automation-related workstation | Jenkins maintenance and monitoring |
| WK05 | `172.16.60.105` | Help Desk L1 workstation | Help Desk user environment |

All five workstations belong to the workstation network associated with VLAN 60.

Their exact operating-system versions, domain membership, local group memberships, and administrative privileges should be recorded after verification.

The workstation roles represent different corporate responsibilities and provide distinct contexts for internal enumeration and attack-path development.

---

## 4. Web Infrastructure

| Hostname | IP address | Network segment | Domain membership | Role |
|---|---|---|---|---|
| WEB01 | Not recorded | VLAN 30 — DMZ | Not domain-joined | DMZ web server |
| WEB02 | Not recorded | VLAN 30 — DMZ | Not domain-joined | DMZ web server |
| WEB03 | `172.16.40.102` | VLAN 40 — Corporate applications | Internal domain network | Internal application server |

WEB01 and WEB02 are isolated from the corporate Active Directory domain and belong to the DMZ.

WEB03 belongs to the internal corporate application network and hosts Mattermost and osTicket.

The separation between the DMZ and corporate application network is an important security boundary. Access between these segments depends on the configured firewall rules and permitted service flows.

---

## 5. Application and Collaboration Services

| System | IP address | Role | Known services | Status |
|---|---|---|---|---|
| Gitea | `172.16.40.100` | Internal source-code and documentation platform | Web interface, SSH, and HTTPS through nginx | Implemented |
| Mattermost | Hosted on WEB03 | Internal collaboration | Team communication and operational channels | Implemented |
| osTicket | Hosted on WEB03 | Help Desk and support | Tickets, support workflows, and organizational context | Implemented |
| MAIL01 | Not recorded | Internal mail server | Postfix, Dovecot, and LDAP-backed authentication | Implemented; LDAP authentication tested |

Gitea is deployed using Docker, with the following documented service ports:

- Web interface: TCP 3000
- SSH: TCP 2222
- HTTPS through nginx

The externally or internally reachable ports depend on the actual host bindings and firewall rules.

Mattermost and osTicket are hosted on WEB03. Their application ports and reverse-proxy configuration should be recorded separately if they change.

MAIL01 uses LDAP authentication against the corporate Active Directory infrastructure. Its IP address and exact mail-service ports should be confirmed from the current deployment.

---

## 6. Database Infrastructure

| Hostname | IP address | Role | Known databases / services | Status |
|---|---|---|---|---|
| MSSQL01 | Not recorded | Application database server | SQL Server 2022 Developer; `RecruitmentDB` | Implemented |
| MSSQL02 | Not recorded | Reporting and operational database server | SQL Server 2022 Developer; `ReportingDB` and certificate inventory-related functionality | Implemented / integration in development |

Both SQL servers use Windows Server Core according to the documented deployment.

The database architecture includes application identities, service accounts, stored procedures, and inter-server relationships.

The following identity is relevant to the reporting environment:

```text
svc_reporting
```

Database permissions, login mappings, linked-server configuration, stored procedures, and trust-related settings belong in `05-databases/`, rather than in this general inventory.

Exact IP addresses and SQL listener ports should be added after checking the current server configurations.

---

## 7. PKI Infrastructure

| Hostname | FQDN | IP address | Role | Status |
|---|---|---|---|---|
| CA01 | `CA01.corp.apexfiend.lab` | `172.16.50.100` | Enterprise Certificate Authority | Implemented |
| CA service | `corp-CA01-CA` | N/A | Certificate Authority identity | Implemented |

The PKI environment uses Active Directory Certificate Services and includes certificate templates, enrollment permissions, and certificate inventory workflows.

The following components are relevant to the current attack-path design:

- `UserTemporary` certificate template
- Certificate-template enumeration from WK03
- Certificate inventory tooling
- CA-related administrative access

Certificate-template enumeration has been tested. The complete certificate-abuse and subsequent escalation chain remains subject to end-to-end validation.

Detailed CA configuration, template properties, and enrollment permissions belong in `06-pki/`.

---

## 8. Jenkins and Automation Infrastructure

| Host / Identity | Address | Role | Status |
|---|---|---|---|
| JENKINS01 | Not recorded | Jenkins automation infrastructure | Implemented; job execution tested |
| `svc_automation$` | N/A | Group Managed Service Account | gMSA configuration tested |
| `m.ortega` | N/A | Jenkins maintenance and monitoring operator | Identity and attack-path integration to verify |
| `m.alvarez` | N/A | Jenkins / CA01 platform administrator; Domain Admin | Configured role; final attack path pending validation |

The Jenkins environment contains the following documented jobs:

```text
Infrastructure-Maintenance
Certificate-Inventory
SQL-Health-Check
Backup-Verification
```

Jobs interact with predefined PowerShell scripts through a build agent mechanism.

The automation workspace is:

```text
C:\ProgramData\ApexAutomation\Jenkins
```

The Jenkins worker uses the label:

```text
apex-automation
```

The automation gMSA is associated with the Jenkins host through:

```text
GG-Jenkins-Hosts
```

The current inventory records the automation configuration separately from the intended privilege-escalation path. Independent testing of Jenkins jobs or the gMSA does not, by itself, validate the entire attack chain.

Detailed job configuration, script behavior, execution permissions, and security implications belong in `07-jenkins/`.

---

## 9. Relevant User and Service Identities

The following identities are included because of their relevance to infrastructure operations or the documented attack-path design.

| Identity | Associated role | Relevant systems / areas | Status |
|---|---|---|---|
| `j.derek` | PKI auditor | WK03, certificate-template enumeration | Identity configured; attack-path integration tracked separately |
| `m.ortega` | Jenkins maintenance and monitoring operator | WK04, Jenkins | Identity configured; attack-path integration to verify |
| `m.alvarez` | Jenkins / CA01 platform administrator and Domain Admin | Jenkins, CA01, Active Directory | Configured role; final chain pending validation |
| `svc_automation$` | Jenkins automation gMSA | Jenkins host and automation workflows | gMSA configuration tested |
| `svc_reporting` | Reporting database service account | SQL reporting infrastructure | Implemented; detailed permissions documented separately |

This table is an inventory of relevant identities, not a complete Active Directory user list.

Passwords, password hashes, private keys, tokens, and other reusable authentication material must not be included in the public repository.

---

## 10. Supporting Infrastructure and Components

Some components are logical services or software deployments rather than independent virtual machines.

| Component | Associated system | Purpose | Status |
|---|---|---|---|
| Docker | Gitea host | Container runtime | Implemented |
| nginx | Gitea host | HTTPS and reverse proxy | Implemented |
| Postfix | MAIL01 | Mail transport | Implemented |
| Dovecot | MAIL01 | Mail access and authentication | Implemented |
| LDAP integration | MAIL01 / Active Directory | Directory-backed mail authentication | Tested |
| Certificate inventory tooling | SQL / PKI infrastructure | Collect certificate and template information | Implemented; workflow tested |
| Jenkins PowerShell build agent | Jenkins host | Map approved jobs to predefined scripts | Implemented; job execution tested |

This section should be updated when new services are added or existing components are moved between hosts.

---

## 11. Inventory Maintenance

The inventory should be updated whenever a system is:

- Added or removed
- Assigned a new IP address
- Moved to another VLAN
- Reconfigured with new services
- Joined to or removed from a domain
- Assigned a different operational role
- Validated as part of an attack path

When documenting a new system, record at least:

1. Hostname and FQDN, where applicable
2. Operating system and version
3. IP address and subnet
4. VLAN or network segment
5. Domain membership
6. Primary function
7. Relevant services and listening ports
8. Implementation and validation status

Configuration should be verified against the running environment rather than inferred from the intended architecture.

---

## Security and Publication Considerations

The public inventory should contain only information necessary to explain the laboratory's architecture.

Do not publish:

- Passwords or password hashes
- Private keys or authentication tokens
- VPN secrets
- Unredacted credential stores
- Reusable authentication material
- Sensitive configuration exports containing secrets

Internal IP addresses, hostnames, and intentionally vulnerable configurations may be documented when appropriate for the lab's public portfolio, provided they do not expose unrelated infrastructure.
