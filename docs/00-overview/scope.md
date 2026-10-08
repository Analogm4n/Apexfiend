\# Apexfiend — Scope



\## Purpose



This document defines the scope of the Apexfiend security laboratory.



Apexfiend is a self-contained environment created for authorized security testing, offensive security training, and research. All systems, identities, applications, and attack paths described in this repository belong to the laboratory unless explicitly stated otherwise.



The scope is intended to establish clear boundaries for both laboratory development and security testing.



\---



\## In-Scope



The following components are considered part of the Apexfiend laboratory scope.



\### Active Directory



The Active Directory environment includes:



\- `apexfiend.lab`

\- `corp.apexfiend.lab`

\- Domain controllers

\- Organizational units

\- Security groups

\- User accounts

\- Computer accounts

\- Service accounts

\- gMSAs

\- Group Policy

\- ACLs and delegated permissions

\- Kerberos authentication

\- NTLM authentication

\- Inter-domain relationships



The security of these components may be assessed from both authenticated and unauthenticated positions depending on the attack scenario.



\### Network Infrastructure



The laboratory network includes:



\- pfSense

\- VLAN segmentation

\- Internal routing

\- Firewall rules

\- VPN access

\- Workstation networks

\- Server networks

\- Application networks

\- PKI networks

\- Management and infrastructure segments



Testing may include network discovery, service enumeration, segmentation analysis, and controlled pivoting between permitted network segments.



\### Windows Systems



Windows systems within the laboratory are in scope, including:



\- Domain controllers

\- Windows workstations

\- Windows Server systems

\- SQL Server hosts

\- Jenkins hosts

\- Certificate Authority infrastructure



Testing may include:



\- Local privilege escalation

\- Credential discovery

\- Authentication attacks

\- Lateral movement

\- Windows service abuse

\- PowerShell-based activity

\- Active Directory attacks

\- Kerberos-based attacks

\- NTLM-based attacks



\### Linux Systems



Linux systems used by the laboratory are also in scope, including servers providing:



\- Web applications

\- Gitea

\- Mattermost

\- osTicket

\- Mail services

\- Supporting infrastructure



Testing may include application enumeration, authentication testing, service configuration analysis, and controlled exploitation.



\### Internal Applications



The following internal applications are part of the laboratory attack surface:



\- Gitea

\- Mattermost

\- osTicket

\- Internal web applications

\- Postfix

\- Dovecot

\- Jenkins

\- Other intentionally deployed laboratory services



These systems may provide information, credentials, access relationships, or attack paths relevant to the wider environment.



\### Database Infrastructure



Microsoft SQL Server infrastructure is in scope, including:



\- MSSQL01

\- MSSQL02

\- RecruitmentDB

\- ReportingDB

\- Application database accounts

\- Service accounts

\- Stored procedures

\- SQL Server permissions

\- Cross-server relationships



Testing may include authentication, authorization, privilege escalation, and trust-boundary analysis.



\### PKI / AD CS



The Enterprise PKI environment is in scope, including:



\- CA01

\- Enterprise Certificate Authority

\- Certificate templates

\- Enrollment permissions

\- Certificate-related configuration

\- Certificate inventory mechanisms

\- Integration between AD CS and Active Directory



Testing may include certificate-template enumeration, configuration analysis, and controlled exploitation of intentionally vulnerable certificate configurations.



\### Automation Infrastructure



Jenkins and its supporting automation infrastructure are in scope, including:



\- Jenkins

\- Jenkins jobs

\- Build permissions

\- Automation workspaces

\- Build agent mechanisms

\- PowerShell automation

\- gMSAs

\- Service execution contexts



Testing may include authorization analysis, job abuse, and privilege-boundary assessment.



\---



\## In-Scope Security Activities



The laboratory may be used for the following activities.



\### Reconnaissance



\- Network discovery

\- DNS enumeration

\- Service discovery

\- Host identification

\- Active Directory enumeration

\- Application enumeration



\### Credential and Authentication Testing



\- Credential discovery

\- Password auditing

\- Kerberos attacks

\- NTLM attacks

\- Authentication relay

\- Service-account assessment

\- Certificate-based authentication testing



\### Privilege Escalation



\- Local privilege escalation

\- Active Directory privilege escalation

\- ACL abuse

\- Delegated permission abuse

\- Service-account abuse

\- gMSA-related attacks

\- RBCD



\### Lateral Movement



\- SMB

\- WinRM

\- RDP

\- SQL Server

\- Kerberos

\- Remote PowerShell

\- Other intentionally exposed management protocols



\### Application Security



\- Authentication testing

\- Authorization testing

\- Information disclosure

\- Configuration analysis

\- Credential exposure

\- Application-to-infrastructure attack paths



\### Post-Exploitation



Controlled post-exploitation activity is permitted when required to demonstrate an attack path or determine impact.



Examples include:



\- Privilege verification

\- Credential and token analysis

\- Access validation

\- Domain-level impact assessment

\- Evidence collection



\---



\## Out-of-Scope



The following are outside the intended scope of the Apexfiend laboratory.



\### External Third-Party Systems



Systems that do not belong to the Apexfiend environment are out of scope.



This includes:



\- Public websites

\- Third-party infrastructure

\- Cloud services not explicitly deployed for the laboratory

\- Internet hosts

\- Other users' systems

\- Production environments



The laboratory should not be used to test external systems without explicit authorization.



\### Real Credentials and Secrets



Real-world credentials and authentication material are out of scope.



The repository must not contain:



\- Real passwords

\- Production credentials

\- API keys

\- Private SSH keys

\- Private certificate keys

\- VPN private keys

\- Authentication tokens

\- Browser session cookies

\- Real NTLM hashes

\- Kerberos tickets from non-laboratory environments



\### Destructive Testing



The following activities should generally be avoided unless specifically required for laboratory development:



\- Irreversible data destruction

\- Intentional filesystem destruction

\- Permanent denial of service

\- Destruction of domain controllers

\- Destruction of the PKI

\- Destruction of the entire database environment



The objective is to demonstrate security impact while keeping the laboratory recoverable.



\### Uncontrolled Malware Activity



Malware development or execution is not a primary objective of Apexfiend.



Any malware-related experimentation should remain isolated from the laboratory and external networks and should not be incorporated into the normal attack paths unless explicitly documented.



\---



\## Evidence and Data Handling



Evidence collected during testing should be sanitized before being committed to the public repository.



Screenshots and command output should be reviewed for:



\- Passwords

\- Hashes

\- Tokens

\- Private keys

\- Personal information

\- Sensitive configuration

\- Internal credentials

\- Authentication material



Where necessary, sensitive values should be replaced with:



```text

<REDACTED>

```



or another clearly identifiable placeholder.



\---



\## Scope Boundaries



The laboratory is intentionally designed to contain multiple trust boundaries.



These include:



```text

Network

&#x20;   ↓

Application

&#x20;   ↓

Identity

&#x20;   ↓

Database

&#x20;   ↓

PKI

&#x20;   ↓

Automation

&#x20;   ↓

Active Directory

```



The security significance of these boundaries is part of the laboratory's intended learning objectives.



A system may therefore be considered secure when evaluated independently but become exploitable when its relationship with another system is taken into account.



\---



\## Authorization



All testing described in this repository is performed against infrastructure intentionally created and controlled for the Apexfiend laboratory.



The laboratory exists specifically to provide an authorized environment for offensive security experimentation.



No authorization should be inferred for testing systems outside this environment.

