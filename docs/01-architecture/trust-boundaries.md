# Trust Boundaries

## 1. Purpose

This document describes the main trust boundaries within the Apexfiend lab. It identifies where network access, identity privileges, application permissions, and administrative control separate one component from another.

The lab is designed to demonstrate that access to one system does not automatically imply access to the rest of the environment. Movement across a boundary depends on the relevant network rules, credentials, permissions, service configuration, and security controls.

The boundaries described here represent the intended architecture and security model. Effective access must be confirmed through configuration review and testing.

## 2. Trust Model Overview

Apexfiend contains several distinct security contexts:

* External development environment and remote access.
* DMZ web servers.
* Corporate workstations and user identities.
* Active Directory domains and domain-level privileges.
* Internal application and database services.
* Public key infrastructure (PKI).
* Jenkins automation and service identities.
* Administrative control over critical infrastructure.

These contexts overlap through legitimate service dependencies, but they should not be treated as a single, uniformly trusted network.

A host's network location, domain membership, user privileges, and ability to administer another service are separate properties. For example, access to a workstation does not necessarily provide administrative access to its domain, and the ability to inspect certificate templates does not necessarily grant permission to enroll in them.

## 3. Network Trust Boundaries

### 3.1 External Environment and VPN

The development workstation uses the external-facing network, with the known address `10.0.100.11`. The VPN client network is `10.0.200.0/24`.

The VPN provides a potential entry point into the lab. Its effective scope depends on the VPN configuration, routes, firewall policy, and the permissions of the connecting identity.

**Boundary conditions:**

* VPN connectivity does not imply unrestricted access to internal networks.
* Access to an internal host does not automatically imply domain privileges.
* Reachability should be established through configuration review or testing, not inferred from routing alone.

### 3.2 DMZ Boundary — VLAN 30

WEB01 and WEB02 reside in VLAN 30 and are not joined to Active Directory.

The DMZ separates these web systems from the corporate application network and domain infrastructure. This separation is intended to limit the consequences of a compromise of a web-facing system.

**Boundary conditions:**

* DMZ hosts do not inherit domain trust through domain membership.
* Any connection from the DMZ to corporate services depends on the applicable firewall and service-level controls.
* A compromise of WEB01 or WEB02 should not be assumed to provide access to WEB03, SQL Server, or domain controllers without an additional path.

The actual degree of isolation depends on the configured firewall rules and the services exposed by each host.

### 3.3 Corporate Workstation Boundary — VLAN 60

VLAN 60 contains five workstations with different operational responsibilities:

| Host | Role                                     | Intended security context       |
| ---- | ---------------------------------------- | ------------------------------- |
| WK01 | Standard corporate workstation           | General corporate user activity |
| WK02 | Web development and DevOps               | Development-related access      |
| WK03 | PKI operations and certificate inventory | Read-oriented PKI operations    |
| WK04 | Jenkins maintenance and automation       | Automation operations           |
| WK05 | Help Desk L1                             | First-line support activity     |

The workstations share a network segment but do not necessarily share the same privileges.

**Boundary conditions:**

* Local administrator access is distinct from domain administrator access.
* Access to a workstation does not automatically grant access to other workstations.
* Operational responsibilities do not establish technical permissions by themselves.
* Effective privileges depend on local group membership, domain groups, ACLs, credentials, and service configuration.

### 3.4 Corporate Application and Data Boundary — VLAN 40

VLAN 40 contains WEB03, Gitea, and the SQL Server infrastructure.

WEB03 belongs to the corporate domain environment. Gitea and the database instances provide application and data services within the same network segment.

**Boundary conditions:**

* Network proximity does not imply shared application credentials or administrative permissions.
* Access to a repository does not automatically grant access to the host running the application.
* Database connectivity does not necessarily imply database ownership, server-level privileges, or operating-system access.
* Access to WEB03 does not automatically provide control over Gitea, MSSQL01, or MSSQL02.

Connections between these components should be evaluated according to their actual protocols, authentication methods, and permissions.

### 3.5 Active Directory Boundaries — VLANs 10 and 20

The forest contains the root domain `apexfiend.lab` and the child domain `corp.apexfiend.lab`.

* VLAN 10 hosts DC01, the root-domain domain controller.
* VLAN 20 hosts DC02 and DC03, the corporate-domain domain controllers.

The domains are part of the same forest, but identities, permissions, and administrative scope must still be evaluated in their actual directory context.

**Boundary conditions:**

* Domain membership does not imply Domain Admin privileges.
* Rights in one domain should not be assumed to confer equivalent rights in another domain.
* Cross-domain access depends on the forest's trust relationships, group memberships, ACLs, and relevant service permissions.
* Control of a domain-joined workstation is not equivalent to control of a domain controller.

The configured trust relationships and effective permissions should be verified against the directory configuration.

### 3.6 PKI Boundary — VLAN 50

CA01 is located in VLAN 50 and runs the enterprise certificate authority named `corp-CA01-CA`.

PKI introduces a security boundary between the ability to inspect certificate configuration, the ability to request a certificate, and the ability to administer the certificate authority.

**Boundary conditions:**

* Read access to certificate templates does not automatically grant enrollment rights.
* Enrollment rights do not automatically grant permission to modify templates or administer the CA.
* The impact of an issued certificate depends on the template configuration, issuance controls, certificate contents, and the identity or privileges represented by the certificate.
* Access to the enrollment web interface does not establish that a user is authorized to obtain a particular certificate.

The `UserTemporary` template is part of the lab's PKI scenario. Its effective permissions and security impact should be assessed from the actual template and CA configuration.

## 4. Identity and Privilege Boundaries

### 4.1 Standard and Operational Identities

Apexfiend includes identities used for ordinary workstation activity, development, help-desk operations, PKI inventory, and automation maintenance.

These roles are intended to model separation of duties. Their actual security scope depends on the permissions assigned to the associated accounts and groups.

A role description should not be treated as proof of effective access. Local group membership, directory ACLs, delegated rights, service permissions, and available credentials must be evaluated independently.

### 4.2 PKI Operations

`j.derek` is intended to perform certificate inventory and read-only template inspection from WK03. The account is not intended to have enrollment or template modification rights.

This boundary distinguishes certificate configuration visibility from the ability to request or control certificates. Any assessment of this boundary should verify the effective permissions on the relevant templates and CA.

### 4.3 Jenkins Operations

`m.ortega` is associated with Jenkins maintenance and monitoring from WK04. The account is not intended to have direct administrative rights over CA01.

This separates routine automation operations from PKI administration. However, indirect access may be possible if Jenkins jobs, execution identities, worker permissions, or directory ACLs provide a path to more privileged resources.

### 4.4 Platform Administration

`m.alvarez` is associated with Jenkins and CA01 platform administration and holds Domain Admin privileges.

This identity represents a high-impact administrative context. Its credentials and administrative access should be protected separately from routine operational accounts.

The existence of a privileged account does not itself establish that a lower-privileged identity can obtain its privileges. Such a transition requires a specific, demonstrable path.

## 5. Automation and Service-Identity Boundary

Jenkins executes predefined PowerShell scripts through its configured automation environment. The lab includes the `svc_automation$` group Managed Service Account (gMSA), associated with `GG-Jenkins-Hosts`.

The security boundary depends on the distinction between:

* Permission to access the Jenkins interface.
* Permission to configure or execute jobs.
* Permission to modify scripts or job definitions.
* Access to the worker or agent execution context.
* Rights held by the identity used to execute a job.
* Rights that identity has on remote systems.

A job that performs a maintenance task may have access beyond the permissions of the user who triggered it, depending on its execution context. The effective impact must be determined from the actual Jenkins authorization model, worker configuration, scripts, and service-account privileges.

The `svc_automation$` identity should not be assumed to have a particular level of access solely because it is a gMSA or is associated with Jenkins.

## 6. Database Security Boundary

MSSQL01 hosts RecruitmentDB, while MSSQL02 hosts ReportingDB.

The database boundary separates network access to a SQL Server instance from authorization within the SQL Server service and its databases.

Relevant security contexts include:

* SQL Server connectivity.
* SQL and Windows authentication.
* Server-level logins and roles.
* Database users and roles.
* Stored procedure execution permissions.
* Database ownership and configuration.
* SQL Server service identity.
* Access from the SQL Server process to the underlying operating system or directory.

A database principal with permission to execute a stored procedure does not necessarily have operating-system access. Conversely, a misconfiguration that crosses the database-to-operating-system boundary may have consequences beyond the database itself.

The actual impact depends on the effective configuration and must be demonstrated with evidence.

## 7. Boundary-Crossing Scenarios

The lab's intended assessment scenarios explore transitions between the following contexts:

1. External development environment or VPN to an internal workstation.
2. Workstation access to application, collaboration, or support information.
3. Application access to database services and their execution contexts.
4. Database or service-account contexts to domain identities.
5. PKI inventory access to certificate enrollment or certificate-related privilege escalation opportunities.
6. Automation maintenance access to Jenkins job execution and worker contexts.
7. Service or directory permissions to higher-privileged infrastructure.
8. Privileged access to CA01 and other critical domain resources.

These are investigation areas, not a guarantee that each transition is exploitable or that every transition has been validated end to end.

## 8. Security Review Checklist

* [ ] Verify the effective pfSense rules between VLANs.
* [ ] Verify the routes and permitted destinations available to VPN clients.
* [ ] Confirm that WEB01 and WEB02 are not domain-joined.
* [ ] Review corporate workstation local and domain privileges.
* [ ] Confirm the root and child domain trust and delegation configuration.
* [ ] Review Gitea, Mattermost, and osTicket authentication and authorization.
* [ ] Review SQL Server logins, database roles, ownership, and service identities.
* [ ] Inspect the `UserTemporary` certificate template and CA permissions.
* [ ] Review Jenkins authorization, job configuration, worker access, and script permissions.
* [ ] Confirm the effective rights of `svc_automation$`.
* [ ] Verify the intended separation between operational identities and privileged administration.
* [ ] Record evidence for each confirmed boundary crossing and each tested restriction.

## 9. Scope and Limitations

This document records the intended trust model for Apexfiend. It is not a complete ACL audit, firewall review, or proof that every trust boundary is correctly enforced.

Where implementation details remain unverified, the intended boundary should be distinguished from observed behavior. Findings should be updated when configuration reviews or controlled tests establish the actual level of access.
