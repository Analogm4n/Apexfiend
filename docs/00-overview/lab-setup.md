\# Apexfiend — Lab Setup



\## Overview



Apexfiend is a virtualized enterprise security laboratory built to simulate a segmented corporate environment.



The laboratory is primarily deployed using VirtualBox, with pfSense providing network routing, VLAN segmentation, and VPN-based access.



The environment contains approximately 20 virtual machines distributed across several functional network segments.



The infrastructure combines Windows and Linux systems to represent domain infrastructure, workstations, application servers, databases, PKI, and automation systems.



\---



\## Virtualization



\### Hypervisor



The laboratory is built using:



\- VirtualBox



Virtual machines are distributed across isolated virtual networks according to their intended role.



The virtualization layer provides the ability to:



\- Create isolated networks

\- Attach systems to multiple network segments

\- Snapshot systems

\- Restore compromised infrastructure

\- Rebuild individual components

\- Test different attack paths without affecting external systems



\---



\## Network Gateway



\### pfSense



pfSense acts as the central network gateway and firewall.



Its primary responsibilities are:



\- Inter-VLAN routing

\- Firewall enforcement

\- VPN access

\- Network segmentation

\- Controlled communication between infrastructure zones



The laboratory is intentionally divided into multiple VLANs rather than operating as a flat network.



\---



\## Network Segmentation



The environment uses separate network segments for different functional areas.



The current design includes:



| VLAN | Primary Function | Example Systems |

|---|---|---|

| VLAN 10 | Root AD infrastructure | DC01 |

| VLAN 20 | Corporate AD infrastructure | DC02, DC03 |

| VLAN 30 | Web infrastructure (DMZ) | Web servers |

| VLAN 40 | Applications / data services | Gitea, WEB03, SQL-related systems |

| VLAN 50 | PKI infrastructure | CA01 |

| VLAN 60 | Workstations / internal access | WK01, WK03, WK04 |



The exact firewall rules and permitted flows are documented separately as part of the network architecture.



The purpose of this segmentation is to create meaningful trust boundaries and require controlled movement between different areas of the environment.



\---



\## Active Directory Forest



The Active Directory environment consists of a multi-domain forest.



```text

apexfiend.lab

└── corp.apexfiend.lab

```



\### Root Domain



```text

apexfiend.lab

```



The root domain contains:



```text

DC01

192.168.10.10

```



DC01 operates as a domain controller for the root domain.



\### Child Domain



```text

corp.apexfiend.lab

```



The child domain contains:



```text

DC02

172.16.20.100



DC03

172.16.20.101

```



DC02 provides domain-controller and PDC/DNS functionality.



DC03 provides Global Catalog and DNS functionality.



The domain infrastructure was validated using standard Active Directory diagnostic procedures, including `dcdiag`.



\---



\## Workstations



Several Windows workstations are distributed through the workstation network.



Important systems include:



\### WK01



```text

172.16.60.101

```



WK01 represents a standard corporate workstation and provides an internal position from which further enumeration can be performed.



\### WK02



```text

172.16.60.102

```



WK02 — Web Developer / DevOps Engineer workstation.



\### WK03



```text

172.16.60.103

```



WK03 is associated with PKI operations and certificate inventory activities.



It provides an example of a workstation used by a technically privileged but non-administrative role.



\### WK04



```text

172.16.60.104

```



WK04 is associated with Jenkins maintenance and automation operations.



It is used in attack paths involving:



\- Jenkins

\- Automation

\- Service accounts

\- Privilege escalation



\### WK05



```text

172.16.60.105

```



WK05 — Help Desk 1 user workstation.



\---



\## Web Infrastructure



The laboratory contains multiple web servers representing different corporate applications.



The web infrastructure includes:



```text

WEB01

WEB02

WEB03

```



WEB03 hosts internal collaboration and support applications, including:



\- Mattermost

\- osTicket



The web environment acts both as an application attack surface and as a source of organizational information.



\---



\## Gitea



Gitea is deployed as an internal source-code and documentation platform.



Current deployment:



```text

172.16.40.100

```



The service is containerized and exposed through:



\- Web interface

\- SSH

\- HTTPS through nginx



Gitea contains internal-style repositories documenting infrastructure and operational information.



One of the main repositories is associated with Web Operations and contains information relating to:



\- Web servers

\- Application ownership

\- Internal infrastructure

\- Operational relationships



The repository is intentionally designed to resemble an internal corporate development environment.



\---



\## Mattermost



Mattermost is deployed as an internal corporate communication platform.



It contains public and private channels representing different departments and operational teams.



Example channels include:



```text

\#welcome

\#it-onboarding

\#it-helpdesk

\#people-ops

\#it-infrastructure

\#web-operations

\#recruitment

```



Private operational channels include areas such as:



```text

\#database-operations

\#pki-operations

\#automation

```



Mattermost provides contextual information about users, departments, infrastructure, and operational procedures.



The service is therefore considered part of the information-discovery layer of the laboratory.



\---



\## osTicket



osTicket provides an internal Help Desk platform.



The simulated organization includes multiple Help Desk roles representing different levels of responsibility.



The service is used to model:



\- User support

\- Internal incidents

\- Operational requests

\- Employee onboarding

\- Technical ownership

\- Historical information



This provides a realistic source of organizational context without requiring every relevant piece of information to be directly exposed through Active Directory enumeration.



\---



\## Mail Infrastructure



MAIL01 provides internal email functionality using:



\- Postfix

\- Dovecot

\- LDAP authentication against Active Directory



The mail environment uses:



```text

@apexfiend.lab

```



while Active Directory identities are associated with:



```text

@corp.apexfiend.lab

```



LDAP queries are directed toward the corporate domain infrastructure.



The mail system is used to represent internal communication and provide additional contextual information during an assessment.



\---



\## Database Infrastructure



The SQL environment consists primarily of:



```text

MSSQL01

MSSQL02

```



\### MSSQL01



MSSQL01 hosts application-oriented database infrastructure, including:



```text

RecruitmentDB

```



The database is associated with the internal Job Portal application.



\### MSSQL02



MSSQL02 hosts:



```text

ReportingDB

```



and supporting infrastructure related to certificate inventory and reporting.



The SQL environment contains deliberately designed relationships between applications, service accounts, stored procedures, and infrastructure.



These relationships are documented separately under the database architecture documentation.



\---



\## PKI Infrastructure



The laboratory contains an Enterprise Active Directory Certificate Services deployment.



\### CA01



```text

172.16.50.100

```



CA01 operates as the Enterprise Certificate Authority for the corporate domain.



The CA is:



```text

corp-CA01-CA

```



The PKI environment includes:



\- Certificate Authority

\- Certificate templates

\- Enrollment permissions

\- Certificate inventory

\- User and machine certificate workflows



Some certificate configurations are intentionally designed to provide security testing scenarios.



\---



\## Certificate Inventory



The laboratory includes a custom certificate inventory component.



The inventory process can query:



\- Active Directory

\- Certificate templates

\- SQL Server

\- CA-related information



The resulting information can be exported to CSV reports for operational analysis.



The inventory workflow is also used to model how legitimate administrative tooling can expose information relevant to an attacker.



\---



\## Jenkins Infrastructure



Jenkins provides the laboratory's automation platform.



The environment contains:



```text

JENKINS01

```



with an automation-oriented configuration.



The Jenkins environment includes approved jobs such as:



```text

Infrastructure-Maintenance

Certificate-Inventory

SQL-Health-Check

Backup-Verification

```



The jobs interact with local PowerShell automation through a dispatcher mechanism.



The dispatcher maps approved build jobs to predefined scripts rather than providing unrestricted command execution.



\---



\## Automation Account



The Jenkins environment uses:



```text

svc\_automation$

```



as a Group Managed Service Account.



The account is associated with the Jenkins host through:



```text

GG-Jenkins-Hosts

```



and is located within an automation-oriented organizational structure.



The automation workspace is:



```text

C:\\ProgramData\\ApexAutomation\\Jenkins

```



The environment uses an `apex-automation` worker label for automation jobs.



\---



\## Service and Administrative Roles



The laboratory contains intentionally differentiated roles.



Examples include:



| Identity | Role |

|---|---|

| `j.derek` | PKI auditor |

| `m.ortega` | Jenkins maintenance / monitoring operator |

| `m.alvarez` | Jenkins / CA01 platform administrator |

| `svc\_automation$` | Jenkins automation gMSA |

| `svc\_reporting` | Reporting database service account |



These roles are intentionally separated so that compromise of one account does not automatically imply compromise of the entire environment.



\---



\## Deployment Philosophy



Apexfiend is not intended to be deployed as a single monolithic image.



The infrastructure is built from independent components so that individual systems can be:



\- Rebuilt

\- Modified

\- Snapshot

\- Restored

\- Reconfigured

\- Tested independently



This also allows individual attack paths to be developed without requiring the entire environment to be recreated.



\---



\## Validation



Infrastructure is validated progressively during development.



Examples of validation include:



\- Active Directory health checks

\- DNS resolution

\- Domain-controller diagnostics

\- LDAP authentication

\- Service connectivity

\- SQL connectivity

\- gMSA validation

\- Certificate-template enumeration

\- Jenkins job execution

\- Internal application access



The laboratory documentation distinguishes infrastructure that has been configured from attack paths that have been fully validated end-to-end.



\---



\## Configuration Management



Configuration information is maintained separately from attack documentation.



The public repository should contain:



\- Sanitized configuration examples

\- Architecture documentation

\- Deployment scripts where appropriate

\- Infrastructure diagrams

\- Non-sensitive service configuration



The repository should not contain:



\- Passwords

\- Private keys

\- Authentication tokens

\- VPN secrets

\- Production credentials

\- Reusable authentication material



Sensitive values should be replaced with explicit placeholders before publication.

