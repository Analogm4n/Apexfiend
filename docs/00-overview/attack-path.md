\# Apexfiend — Attack Path



\## Overview



Apexfiend is designed around multi-stage attack paths rather than isolated vulnerabilities.



The primary attack chain demonstrates how individually limited compromises can be combined across different enterprise services to achieve progressively higher levels of access.



The intended path crosses several security boundaries:



```text

Initial Access

&#x20;     │

&#x20;     ▼

Internal Enumeration

&#x20;     │

&#x20;     ▼

Web / Gitea / Mattermost / Mail

&#x20;     │

&#x20;     ▼

SQL Infrastructure

&#x20;     │

&#x20;     ▼

MSSQL01$

&#x20;     │

&#x20;     ▼

MSSQL02$

&#x20;     │

&#x20;     ▼

j.derek

&#x20;     │

&#x20;     ▼

WK03

&#x20;     │

&#x20;     ▼

AD CS / UserTemporary

&#x20;     │

&#x20;     ▼

NTLM Relay

&#x20;     │

&#x20;     ▼

m.ortega

&#x20;     │

&#x20;     ▼

WK04

&#x20;     │

&#x20;     ▼

Jenkins

&#x20;     │

&#x20;     ▼

svc\_automation$

&#x20;     │

&#x20;     ▼

WriteDACL

&#x20;     │

&#x20;     ▼

RBCD

&#x20;     │

&#x20;     ▼

CA01

&#x20;     │

&#x20;     ▼

Domain Administrative Impact

&#x20;     │

&#x20;     ▼

ExtraSid (DC02$ → DC01$)

```



This document provides a high-level representation of the attack path. Detailed enumeration, exploitation procedures, evidence, and remediation are documented separately.



\---



\## Attack Path Philosophy



The attack path is intentionally designed so that no single weakness provides immediate domain compromise.



Instead, the attacker must:



1\. Obtain an initial foothold.

2\. Understand the internal environment.

3\. Correlate information from multiple services.

4\. Identify relationships between users, systems, and service accounts.

5\. Abuse delegated privileges and authentication mechanisms.

6\. Move between network segments.

7\. Chain several otherwise limited security weaknesses.

8\. Reach a high-privilege administrative position.



This reflects the type of reasoning required during an internal Active Directory assessment.



\---



\## Stage 1 — Initial Access



The first stage provides access to the internal environment through the simulated external/development workstation context.



The initial position is intended to provide:



\- VPN access

\- Basic internal network connectivity

\- Access to the workstation network

\- Limited visibility into internal services



The objective is not to provide administrative privileges but to establish a realistic starting position for an internal assessment.



\*\*Status:\*\* Implemented.



\---



\## Stage 2 — Internal Enumeration



Once inside the environment, the attacker performs manual enumeration of the available network and Active Directory infrastructure.



Relevant targets include:



\- Domain controllers

\- Workstations

\- Web servers

\- Gitea

\- Mattermost

\- osTicket

\- Mail infrastructure

\- SQL servers

\- PKI infrastructure

\- Jenkins



The laboratory intentionally rewards understanding the environment rather than relying exclusively on automated BloodHound or PowerView collection.



The objective is to establish:



```text

Users

&#x20;  ↓

Groups

&#x20;  ↓

Systems

&#x20;  ↓

Applications

&#x20;  ↓

Trust relationships

&#x20;  ↓

Administrative responsibilities

```



\*\*Status:\*\* Implemented.



\---



\## Stage 3 — Information Discovery



Several internal applications provide complementary information.



\### Gitea



Internal repositories provide infrastructure and operational information, including:



\- Server names

\- Application ownership

\- Service relationships

\- Operational documentation



\### Mattermost



Internal communication provides contextual information about:



\- Employees

\- Teams

\- Responsibilities

\- Infrastructure

\- Operational procedures



\### osTicket



Help Desk information can reveal:



\- User roles

\- Technical ownership

\- Historical incidents

\- Operational requests

\- Employee onboarding information



\### Mail



Internal email provides another source of organizational context and can correlate identities, services, and operational responsibilities.



The purpose of these services is not to directly disclose the attack path, but to provide realistic information that can be correlated with technical enumeration.



\*\*Status:\*\* Implemented.



\---



\## Stage 4 — SQL Infrastructure



The next stage involves the relationship between the web/application infrastructure and the SQL environment.



The relevant infrastructure includes:



```text

WEB03

&#x20;  │

&#x20;  ▼

MSSQL01

&#x20;  │

&#x20;  ▼

MSSQL02

```



MSSQL01 hosts application-oriented data such as `RecruitmentDB`, while MSSQL02 contains `ReportingDB` and additional operational infrastructure.



The SQL environment is designed to demonstrate how application identities, database permissions, and Windows-integrated authentication can become relevant to an Active Directory attack path.



The intended result of this stage is access to the SQL environment and, ultimately, a position associated with:



```text

MSSQL02$

```



\*\*Status:\*\* Implemented / partially validated depending on the specific attack-path integration.



\---



\## Stage 5 — MSSQL02$ to j.derek



The attack path then transitions from the database infrastructure into the PKI environment.



The intended relationship allows the attacker to leverage the compromised SQL infrastructure to reach the identity:



```text

j.derek

```



`j.derek` represents a PKI auditing role rather than a highly privileged domain administrator.



His access is intentionally limited.



Relevant privileges include the ability to inspect certificate-template information from WK03 without having direct certificate enrollment or template modification permissions.



This creates an intermediate identity that is useful for reconnaissance without immediately providing administrative control.



\*\*Status:\*\* Designed; integration being validated.



\---



\## Stage 6 — PKI Discovery from WK03



Using the access associated with `j.derek`, the attacker reaches:



```text

WK03

172.16.60.100

```



From this position, certificate-template information can be enumerated.



The relevant PKI component is the Enterprise CA:



```text

CA01

172.16.50.100

```



with the CA:



```text

corp-CA01-CA

```



One of the relevant certificate templates is:



```text

UserTemporary

```



The important characteristic of this stage is that the attacker does not need direct administrative control of the CA.



Instead, the attack relies on understanding the permissions and authentication relationships surrounding certificate enrollment and related infrastructure.



\*\*Status:\*\* Certificate-template enumeration tested; exploitation chain in development.



\---



\## Stage 7 — Certificate Abuse / NTLM Relay



The next stage is intended to combine the PKI configuration with an NTLM relay opportunity.



The objective is to obtain or relay authentication in a way that provides access associated with:



```text

m.ortega

```



`m.ortega` is intentionally not a CA administrator.



His role is:



```text

Jenkins Maintenance and Monitoring Operator

```



and is associated primarily with:



```text

WK04

```



This distinction is important to the attack design.



The compromise of `m.ortega` does not directly provide domain administrative privileges. Instead, it provides access to another operational layer of the environment.



\*\*Status:\*\* Designed; end-to-end validation pending.



\---



\## Stage 8 — m.ortega → WK04



Using the compromised operational identity, the attacker reaches:



```text

WK04

```



WK04 is associated with Jenkins maintenance and automation activities.



The objective is to identify how a relatively limited workstation/operator role interacts with the Jenkins infrastructure.



The attacker is expected to correlate:



```text

m.ortega

&#x20;   ↓

WK04

&#x20;   ↓

Jenkins

```



rather than treating Jenkins as an isolated web application.



\*\*Status:\*\* Designed.



\---



\## Stage 9 — Jenkins



The Jenkins environment provides the next privilege boundary.



The relevant jobs include:



```text

Infrastructure-Maintenance

Certificate-Inventory

SQL-Health-Check

Backup-Verification

```



Jenkins uses an automation-oriented execution model in which approved jobs invoke predefined PowerShell scripts through a build agent.



The important security relationship is the combination of:



\- Jenkins permissions

\- Local execution context

\- Automation scripts

\- The Jenkins host

\- The automation service identity



The intended result is execution within the Jenkins automation context.



\*\*Status:\*\* Jenkins infrastructure and job execution tested; attack-path integration in development.



\---



\## Stage 10 — svc\_automation$



Jenkins operates with the Group Managed Service Account:



```text

svc\_automation$

```



The account is associated with the Jenkins host through:



```text

GG-Jenkins-Hosts

```



and operates within the automation infrastructure.



The gMSA configuration has been validated independently.



The attack path uses the Jenkins execution context to pivot from the compromised operator environment into the privileges and relationships associated with the automation account.



\*\*Status:\*\* gMSA configuration tested; full attack-path integration pending.



\---



\## Stage 11 — WriteDACL



The automation identity is intended to have a delegated permission that becomes significant from an offensive perspective.



The relevant relationship is:



```text

svc\_automation$

&#x20;       │

&#x20;       ▼

&#x20;  WriteDACL

&#x20;       │

&#x20;       ▼

Server-related OU / computer object

```



The permission does not represent unrestricted domain administration.



Instead, it provides the ability to modify access-control relationships in a controlled part of Active Directory.



This is another deliberate privilege boundary in the attack chain.



\*\*Status:\*\* Designed; end-to-end exploitation pending.



\---



\## Stage 12 — Resource-Based Constrained Delegation



The delegated permission is intended to enable an RBCD-based attack against:



```text

CA01$

```



The resulting relationship can be represented conceptually as:



```text

svc\_automation$

&#x20;       │

&#x20;       ▼

&#x20;  WriteDACL

&#x20;       │

&#x20;       ▼

&#x20;   CA01$

&#x20;       │

&#x20;       ▼

&#x20;     RBCD

```



The purpose of this stage is to demonstrate how an apparently limited Active Directory delegation can become significantly more powerful when combined with control over a computer object.



The attack does not depend on compromising the CA through a direct administrative login.



Instead, the attacker abuses Active Directory delegation and authentication mechanisms to obtain a privileged security context on the CA host.



\*\*Status:\*\* Designed; end-to-end validation pending.



\---



\## Stage 13 — CA01



The final infrastructure target is:



```text

CA01

172.16.50.100

```



CA01 is the Enterprise Certificate Authority for the corporate domain.



Compromise of the CA represents a major escalation because the system is part of the organization's authentication and certificate infrastructure.



At this point the attack has crossed several independent trust boundaries:



```text

Application

&#x20;  ↓

Database

&#x20;  ↓

User identity

&#x20;  ↓

Workstation

&#x20;  ↓

PKI

&#x20;  ↓

Jenkins

&#x20;  ↓

Automation identity

&#x20;  ↓

Active Directory delegation

&#x20;  ↓

Certificate Authority

```



\*\*Status:\*\* Target state designed; complete chain pending validation.



\---



\## Stage 14 — Domain Administrative Impact



The final objective is to demonstrate domain-level administrative impact.



The intended privileged relationship includes:



```text

m.alvarez

```



who has the combined role of:



\- Jenkins / CA01 Platform Administrator

\- Domain Administrator



The final attack path is therefore intended to demonstrate how compromise of multiple operational layers can eventually reach a domain-administrative security boundary.



The goal is not to demonstrate that `m.alvarez` has a weak password or that Domain Admin is directly exposed.



Instead, the intended finding is that a sequence of individually limited permissions can combine into a domain-level compromise.



\*\*Status:\*\* Designed; end-to-end validation pending.



\---



\## Stage 15 — ExtraSid: DC02$ → DC01$



The attack chain does not stop at the corporate child domain.



Once domain-administrative impact has been achieved within:



```text

corp.apexfiend.lab

```



the attacker is intended to extend that compromise into the forest root domain:



```text

apexfiend.lab

```



via the `SIDHistory` (ExtraSid) attribute, exploiting the inherent inter-domain trust within the same Active Directory forest.



The relevant relationship is:



```text

DC02$

&#x20; │

&#x20; ▼

ExtraSid

&#x20; │

&#x20; ▼

DC01$

```



Because `corp.apexfiend.lab` is a child domain of `apexfiend.lab`, a forged Kerberos ticket carrying an Enterprise Admins SID from the root domain — injected via `SIDHistory` and signed with the child domain's krbtgt key obtained from DC02 — can be used to gain administrative access to the forest root, represented by:



```text

DC01

```



This stage demonstrates that compromise of a child domain is, in practice, compromise of the entire forest, and closes the attack chain at the highest possible level of Active Directory impact.



\*\*Status:\*\* Designed; end-to-end validation pending.



\---



\## Attack Path Summary



The complete intended chain is:



```text

Initial Access

&#x20;     │

&#x20;     ▼

Internal Enumeration

&#x20;     │

&#x20;     ▼

Gitea / Mattermost / osTicket / Mail

&#x20;     │

&#x20;     ▼

Web / SQL Infrastructure

&#x20;     │

&#x20;     ▼

MSSQL02$

&#x20;     │

&#x20;     ▼

j.derek

&#x20;     │

&#x20;     ▼

WK03

&#x20;     │

&#x20;     ▼

AD CS / UserTemporary

&#x20;     │

&#x20;     ▼

NTLM Relay

&#x20;     │

&#x20;     ▼

m.ortega

&#x20;     │

&#x20;     ▼

WK04

&#x20;     │

&#x20;     ▼

Jenkins

&#x20;     │

&#x20;     ▼

svc\_automation$

&#x20;     │

&#x20;     ▼

WriteDACL

&#x20;     │

&#x20;     ▼

CA01$

&#x20;     │

&#x20;     ▼

RBCD

&#x20;     │

&#x20;     ▼

CA01

&#x20;     │

&#x20;     ▼

Domain Administrative Impact

&#x20;     │

&#x20;     ▼

ExtraSid (DC02$ → DC01$)

```



\---



\## Attack Path Status



| Stage | Component / Identity | Status |

|---|---|---|

| Initial access | VPN / workstation network | Implemented |

| Internal enumeration | AD / network infrastructure | Implemented |

| Information discovery | Gitea / Mattermost / osTicket / Mail | Implemented |

| SQL path | WEB03 / MSSQL01 / MSSQL02 | Implemented / partially validated |

| SQL → identity | `MSSQL02$` → `j.derek` | Implemented |

| PKI enumeration | WK03 / certificate templates | Tested |

| Certificate abuse | `UserTemporary` | In development |

| NTLM relay | → `m.ortega` | Designed |

| Operator access | `m.ortega` → WK04 | Designed |

| Jenkins | Jenkins automation | Tested independently |

| Automation identity | `svc\_automation$` | Tested independently |

| Delegation | WriteDACL | Designed |

| RBCD | `CA01$` | Designed |

| CA compromise | CA01 | Designed |

| Domain impact | `m.alvarez` / Domain Admin | Designed |

| Forest compromise | ExtraSid `DC02$` → `DC01$` | Designed |



\---



\## Security Objectives Demonstrated



The attack path is intended to demonstrate several common enterprise security problems.



\### Excessive trust between systems



Applications, databases, workstations, and infrastructure services can create unintended paths between security boundaries.



\### Delegated privileges



A user or service account does not need to be a Domain Administrator for its permissions to become security-critical.



\### PKI security



Certificate services can introduce significant attack paths when template, enrollment, and authentication configurations are not carefully controlled.



\### NTLM authentication



Legacy authentication mechanisms can create relay opportunities when combined with accessible services and inappropriate protocol protections.



\### Automation security



Jenkins and other automation systems can become privilege-escalation boundaries when their execution context has broader privileges than their operators require.



\### Active Directory delegation



Permissions such as `WriteDACL` can become dangerous when applied to security-sensitive objects or organizational structures.



\### Forest trust boundaries



Domain compromise within a child domain is not necessarily contained to that domain. Inter-domain trust mechanisms within a single forest, such as `SIDHistory`, can extend a compromise to the forest root.



\### Chained compromise



The most important characteristic of the laboratory is the combination of these weaknesses.



No individual stage is intended to represent the complete compromise.



The security impact emerges from the relationship between multiple stages.



\---



\## Validation Philosophy



Apexfiend distinguishes between:



\- \*\*Implemented\*\* — The infrastructure or feature has been configured.

\- \*\*Tested\*\* — The component or technique has been independently verified.

\- \*\*In development\*\* — The attack-path integration is being implemented or refined.

\- \*\*Designed\*\* — The intended relationship exists as part of the laboratory architecture but has not yet been fully validated.

\- \*\*Validated end-to-end\*\* — The complete sequence has been successfully reproduced from the preceding stage to the target objective.



This distinction is maintained to prevent the public documentation from presenting planned attack paths as completed compromises.



Detailed evidence and technical procedures will be documented separately as individual attack-path components are validated.

