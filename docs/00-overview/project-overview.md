\# Apexfiend — Project Overview



\## Overview



Apexfiend is a self-built enterprise Active Directory security lab designed to simulate a small corporate environment and provide a realistic platform for practicing internal penetration testing and offensive security techniques.



Rather than relying on isolated vulnerable machines, the lab connects identity infrastructure, network segmentation, internal applications, databases, PKI, automation platforms, and user workstations into a single environment. The objective is to create attack paths where individual weaknesses, excessive privileges, trust relationships, and operational dependencies can be chained together to achieve progressively higher levels of access.



The environment is primarily focused on Active Directory security, internal network penetration testing, and attack-path development, with particular emphasis on enterprise identity infrastructure and Windows-based environments.



\## Design Philosophy



Apexfiend was built around three principles:



\### 1. Realistic enterprise relationships



Systems and accounts are designed to have a purpose within the simulated organization.



For example, users may belong to specific departments, service accounts may be associated with applications, and administrative access may be distributed between different teams rather than being concentrated in a single privileged account.



This allows the lab to model the relationships that commonly exist between:



\- Users and workstations

\- IT teams and infrastructure

\- Applications and databases

\- Automation systems and service accounts

\- PKI administrators and certificate infrastructure

\- Domain identities and server resources



\### 2. Chained attack paths



The lab is designed so that compromise of a single system does not necessarily provide immediate domain-wide access.



Instead, the attacker is expected to:



1\. Obtain an initial foothold.

2\. Enumerate the internal environment.

3\. Identify users, systems, and trust relationships.

4\. Discover additional information through internal services.

5\. Abuse weaknesses or excessive permissions.

6\. Move laterally between systems.

7\. Escalate privileges.

8\. Reach increasingly sensitive infrastructure.



This approach is intended to reproduce the reasoning required during an internal penetration test rather than treating each machine as an independent challenge.



\### 3. Information-driven exploitation



Not every important piece of information is intended to be directly exposed through automated enumeration.



The environment contains internal documentation, repositories, communication platforms, email, and operational information that can help an attacker understand how the organization operates.



This creates a distinction between:



\- Discovering that a system exists

\- Understanding what the system is used for

\- Identifying which users operate it

\- Determining how those systems trust each other

\- Identifying how that information can be used during an attack



The lab therefore encourages manual enumeration and contextual analysis in addition to automated tooling.



\## Environment



The current environment contains approximately 20 virtual machines distributed across several network segments.



The infrastructure includes:



\- A multi-domain Active Directory forest

\- Windows Server domain controllers

\- Windows workstations

\- pfSense-based network routing and segmentation

\- Internal web servers

\- Gitea

\- Mattermost

\- osTicket

\- Postfix and Dovecot (Outlook)

\- Microsoft SQL Server

\- Enterprise Active Directory Certificate Services (AD CS)

\- Jenkins

\- Service accounts and gMSAs

\- Internal automation infrastructure



The Active Directory forest is based around:



```text

apexfiend.lab

└── corp.apexfiend.lab

```



The environment is segmented using multiple VLANs to represent different functional areas of a corporate network.



\## Core Security Areas



The laboratory focuses on several areas of offensive security:



\- Active Directory enumeration

\- Windows authentication and authorization

\- Kerberos

\- NTLM

\- Active Directory ACLs and delegated permissions

\- Lateral movement

\- SQL Server security

\- Service account abuse (including gMSA)

\- AD CS and certificate-based attacks

\- NTLM relay

\- Jenkins security

\- Resource-Based Constrained Delegation (RBCD)

\- Network segmentation and pivoting

\- Post-exploitation

\- Attack-path analysis



\## Infrastructure as Part of the Attack Surface



Apexfiend intentionally treats infrastructure and operational systems as part of the attack surface.



Applications such as Gitea, Mattermost, osTicket, email, SQL Server, and Jenkins are not necessarily intended to contain a single critical vulnerability. Instead, they provide information, credentials, access relationships, or execution paths that can become relevant later in an attack.



This allows the same identity or piece of information to have different significance depending on the attacker's current position within the environment.



\## Security Research and Training Environment



Apexfiend is a controlled laboratory created for security research, training, and experimentation.



All systems, accounts, services, vulnerabilities, and attack paths are intentionally created within the laboratory. The environment is isolated from production infrastructure and is not intended to be used against third-party systems.



The public documentation is sanitized to avoid exposing passwords, private keys, authentication material, tokens, or other sensitive information.



\## Current Development State



Apexfiend is an evolving project rather than a fixed challenge.



The underlying infrastructure has been progressively expanded from a basic Active Directory environment into a larger enterprise-style network incorporating databases, internal applications, PKI, and automation.



Some attack paths are fully implemented and tested, while others are still being integrated and validated end-to-end.



Documentation therefore distinguishes between:



| Status | Meaning |

|---|---|

| \*\*Implemented\*\* | Infrastructure or functionality has been configured. |

| \*\*Tested\*\* | The relevant functionality has been successfully validated. |

| \*\*In development\*\* | Implementation exists but requires further testing or refinement. |

| \*\*Planned\*\* | Part of the design has been defined but has not yet been implemented. |



This distinction is maintained to keep the project documentation technically accurate.

