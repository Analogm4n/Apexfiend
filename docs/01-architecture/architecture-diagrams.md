# Architecture Diagrams

## 1. Purpose

This document provides visual representations of the Apexfiend architecture using Mermaid diagrams.

The diagrams summarize the network segments, Active Directory structure, major infrastructure dependencies, and selected security boundaries. They are intended to complement the detailed descriptions in:

* `network-architecture.md`
* `system-inventory.md`
* `firewall-and-routing.md`
* `infrastructure-dependencies.md`
* `trust-boundaries.md`

The diagrams represent the documented design. A connection between components indicates an architectural relationship or potential dependency, not necessarily unrestricted network access or a fully validated attack path.

## 2. Network Topology

The following diagram shows the main network segments and their known components. pfSense provides the network gateway and firewall role. Actual routing and permitted traffic depend on its configuration.

```mermaid
flowchart TB
    EXT["External Network<br/>10.0.100.0/24"]
    DEV["Development Workstation<br/>10.0.100.11"]
    PFS["pfSense<br/>WAN: 10.0.100.6"]
    VPN["VPN Clients<br/>10.0.200.0/24"]

    subgraph V10["VLAN 10 — Root Active Directory"]
        DC01["DC01<br/>192.168.10.10"]
    end

    subgraph V20["VLAN 20 — Corporate Active Directory"]
        DC02["DC02<br/>172.16.20.100<br/>PDC Emulator / DNS"]
        DC03["DC03<br/>172.16.20.101<br/>Global Catalog / DNS"]
    end

    subgraph V30["VLAN 30 — DMZ"]
        WEB01["WEB01<br/>IP not recorded<br/>Not domain-joined"]
        WEB02["WEB02<br/>IP not recorded<br/>Not domain-joined"]
    end

    subgraph V40["VLAN 40 — Corporate Applications and Data"]
        WEB03["WEB03<br/>172.16.40.102"]
        GITEA["Gitea<br/>172.16.40.100"]
        SQL01["MSSQL01<br/>IP not recorded"]
        SQL02["MSSQL02<br/>IP not recorded"]
    end

    subgraph V50["VLAN 50 — PKI"]
        CA01["CA01<br/>172.16.50.100<br/>corp-CA01-CA"]
    end

    subgraph V60["VLAN 60 — Workstations"]
        WK01["WK01<br/>172.16.60.101"]
        WK02["WK02<br/>172.16.60.102"]
        WK03["WK03<br/>172.16.60.103"]
        WK04["WK04<br/>172.16.60.104"]
        WK05["WK05<br/>172.16.60.105"]
    end

    MAIL["MAIL01<br/>IP not recorded"]
    JENKINS["Jenkins<br/>IP not recorded"]

    EXT --- DEV
    EXT --- PFS
    DEV -. "Remote access, subject to configuration" .-> VPN
    VPN -. "Permitted routes only" .-> PFS

    PFS --- V10
    PFS --- V20
    PFS --- V30
    PFS --- V40
    PFS --- V50
    PFS --- V60

    DC02 -. "LDAP / directory dependency" .-> MAIL
    WEB03 --- MAIL
    WEB03 --- JENKINS
```

**Diagram notes**

* VLANs are shown as logical segments connected through the firewall. The diagram does not imply that every segment can communicate freely with every other segment.
* The exact attachment of MAIL01 and Jenkins to their network interfaces should be updated if their final placement differs from the current design.
* Server addresses marked as not recorded should be filled in after verification.
* The VPN relationship represents remote access at an architectural level; the actual client routes and firewall permissions must be checked separately.

## 3. Active Directory and Identity Structure

This diagram represents the forest and the major identity-related infrastructure. It does not attempt to show every organizational unit, group, or access control entry.

```mermaid
flowchart TB
    FOREST["Active Directory Forest<br/>apexfiend.lab"]

    ROOT["Root Domain<br/>apexfiend.lab"]
    CHILD["Child Domain<br/>corp.apexfiend.lab"]

    DC01["DC01<br/>Root Domain Controller"]
    DC02["DC02<br/>PDC Emulator / DNS"]
    DC03["DC03<br/>Global Catalog / DNS"]

    WORKSTATIONS["Corporate Workstations<br/>WK01–WK05"]
    WEB03["WEB03<br/>Corporate Application Host"]
    CA01["CA01<br/>Enterprise CA"]

    FOREST --> ROOT
    FOREST --> CHILD

    ROOT --- DC01
    CHILD --- DC02
    CHILD --- DC03

    CHILD -. "Domain membership / configured access" .-> WORKSTATIONS
    CHILD -. "Corporate domain environment" .-> WEB03
    CHILD -. "Directory-integrated PKI" .-> CA01
```

**Diagram notes**

* DC01 belongs to the root domain. DC02 and DC03 belong to the corporate child domain.
* The diagram indicates domain placement, not the exact replication, trust, or authentication traffic.
* WEB01 and WEB02 are intentionally omitted from the domain-membership relationships because they are not domain-joined.
* The effective privileges of users and service accounts depend on their actual memberships and permissions.

## 4. Application and Service Dependencies

This diagram highlights the principal application and infrastructure relationships. Some connections represent dependencies to validate rather than confirmed direct integrations.

```mermaid
flowchart LR
    USERS["Users and Workstations"]

    GITEA["Gitea<br/>Source Repositories"]
    WEB03["WEB03<br/>Mattermost / osTicket"]
    MAIL["MAIL01<br/>Postfix / Dovecot"]

    SQL01["MSSQL01<br/>RecruitmentDB"]
    SQL02["MSSQL02<br/>ReportingDB"]

    AD["Active Directory<br/>DNS / LDAP"]
    CA["CA01<br/>Certificate Services"]

    JENKINS["Jenkins"]
    WORKER["Automation Worker<br/>apex-automation"]
    GMSA["svc_automation$<br/>gMSA"]

    USERS --> GITEA
    USERS --> WEB03

    MAIL -. "LDAP authentication" .-> AD
    WEB03 -. "Identity or mail integration, if configured" .-> AD
    WEB03 -. "Database dependency, if configured" .-> SQL01
    WEB03 -. "Database dependency, if configured" .-> SQL02

    JENKINS --> WORKER
    WORKER --> GMSA

    JENKINS -. "Certificate inventory job" .-> CA
    JENKINS -. "SQL health-check job" .-> SQL01
    JENKINS -. "SQL health-check job" .-> SQL02

    AD -. "Directory-integrated PKI" .-> CA
```

**Diagram notes**

* The LDAP relationship between MAIL01 and Active Directory was tested during setup.
* Mattermost and osTicket are hosted on WEB03. Their additional database, mail, or directory integrations should be confirmed from the deployed configuration.
* Jenkins includes jobs for certificate inventory and SQL health checks. The actual targets, execution identities, and required permissions should be confirmed from the job definitions and scripts.
* The diagram does not imply that the gMSA has administrative rights over the CA or SQL Server.

## 5. Security Boundary Overview

The following diagram groups components by their principal security context.

```mermaid
flowchart TB
    EXT["External Environment"]
    VPN["VPN Entry Point"]

    subgraph DMZ["DMZ — VLAN 30"]
        WEB01["WEB01"]
        WEB02["WEB02"]
    end

    subgraph CORP["Corporate Workstations and Applications"]
        WK["WK01–WK05"]
        WEB03["WEB03"]
        GITEA["Gitea"]
        SQL["MSSQL01 / MSSQL02"]
    end

    subgraph IDENTITY["Active Directory"]
        DC01["DC01 — Root Domain"]
        DC02["DC02 — Corporate Domain"]
        DC03["DC03 — Corporate Domain"]
    end

    subgraph CRITICAL["Privileged Infrastructure"]
        CA["CA01 — PKI"]
        JENKINS["Jenkins Automation"]
        GMSA["svc_automation$"]
    end

    EXT -. "Remote access path" .-> VPN
    VPN -. "Permitted access only" .-> WK

    WEB01 -. "Restricted cross-segment access" .-> CORP
    WEB02 -. "Restricted cross-segment access" .-> CORP

    WK -. "Authenticated directory access" .-> IDENTITY
    WEB03 -. "Configured identity dependencies" .-> IDENTITY
    SQL -. "Service and identity dependencies" .-> IDENTITY

    JENKINS --> GMSA
    JENKINS -. "Configured automation tasks" .-> CA
    JENKINS -. "Configured automation tasks" .-> SQL
```

**Diagram notes**

* The DMZ, corporate environment, Active Directory, and privileged infrastructure are separate security contexts.
* Dashed lines denote conceptual or conditional relationships, not verified firewall permissions.
* The diagram intentionally does not show a direct route from every lower-trust component to every higher-trust component.
* A valid security assessment should establish the exact conditions required to cross each boundary.

## 6. High-Level Assessment Path

The following is a high-level representation of the intended assessment scenario. It is not a substitute for the detailed attack-path documentation, and it does not assert that the entire chain has been validated end to end.

```mermaid
flowchart TD
    ENTRY["Initial Access<br/>VPN / Internal Entry"]
    ENUM["Internal Enumeration"]
    APPS["Application and Collaboration<br/>Information Discovery"]
    DB["Database Access<br/>MSSQL01 / MSSQL02"]
    SQLCTX["SQL Server Service Context"]
    PKIUSER["PKI Operations Context<br/>j.derek / WK03"]
    ADCS["Certificate Template<br/>and Enrollment Assessment"]
    OPS["Automation Operations<br/>m.ortega / WK04"]
    JENKINS["Jenkins Job and Worker Context"]
    GMSA["svc_automation$"]
    DIRECTORY["Directory Permission Assessment"]
    CA01["CA01 / Critical Infrastructure"]
    IMPACT["Potential Privileged Impact"]

    ENTRY --> ENUM
    ENUM --> APPS
    APPS --> DB
    DB --> SQLCTX
    SQLCTX --> PKIUSER
    PKIUSER --> ADCS
    ADCS --> OPS
    OPS --> JENKINS
    JENKINS --> GMSA
    GMSA --> DIRECTORY
    DIRECTORY --> CA01
    CA01 --> IMPACT

    classDef unverified stroke-dasharray: 5 5
    class ENTRY,ENUM,APPS,DB,SQLCTX,PKIUSER,ADCS,OPS,JENKINS,GMSA,DIRECTORY,CA01,IMPACT unverified
```

**Diagram notes**

* Dashed styling indicates that the diagram is a conceptual scenario requiring validation, not that every step has been proven.
* The exact transition conditions, evidence, and status of each step belong in `attack-path.md`.
* Do not publish credentials, tokens, hashes, private keys, or other reusable authentication material in these diagrams or their supporting documentation.
* Update the diagram if testing changes the order of the path or shows that a transition is not viable.

## 7. Diagram Maintenance

When updating these diagrams:

* Keep hostnames, domain names, VLAN assignments, and confirmed IP addresses consistent with `system-inventory.md`.
* Mark unconfirmed addresses as `IP not recorded` rather than guessing.
* Distinguish domain membership from mere network connectivity.
* Use dashed lines for conceptual, conditional, or unverified relationships.
* Avoid representing a possible attack path as a confirmed path unless evidence supports it.
* Update the dependency diagrams when application integrations or automation jobs change.
* Keep detailed firewall rules and service-specific permissions in their respective documentation.

Mermaid source should remain in the Markdown files so that diagrams can be reviewed and version-controlled alongside the architecture documentation.
