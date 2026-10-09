# Apexfiend — Network Architecture

## Overview

Apexfiend uses a segmented virtual network architecture designed to simulate an enterprise environment with separate infrastructure, application, data and workstation zones.

The laboratory is deployed primarily through VirtualBox, with pfSense providing network routing, firewall enforcement and VPN-based access.

Network segmentation is a central component of the environment. Systems are distributed across separate VLANs according to their operational roles, creating boundaries that can be evaluated during internal security assessments.

The architecture supports testing scenarios involving:

* Network and service enumeration
* Inter-VLAN connectivity
* Active Directory authentication
* Lateral movement between systems
* Access to application and database infrastructure
* PKI and administrative services
* Network pivoting and segmentation controls

The laboratory is intended to remain isolated from unrelated networks and external systems unless a specific connection is deliberately configured for a testing purpose.

---

## Network Topology

The laboratory uses pfSense as its central gateway and firewall, connecting several logically separated network segments.

```text
                   Development / External Network
                         10.0.100.0/24
                                |
                            pfSense
                         WAN: 10.0.100.6
                                |
                    VPN: 10.0.200.0/24
                                |
                       VLAN 60 — Workstations
                       WK01 / WK02 / WK03
                            WK04 / WK05
                                |
             +------------------+------------------+
             |                  |                  |
       VLAN 10 — AD       VLAN 20 — AD       VLAN 30 — DMZ
         Root Domain      Corporate Domain     WEB01 / WEB02
             |                  |
            DC01             DC02 / DC03

             +------------------+------------------+
                                |
                    +-----------+-----------+
                    |                       |
             VLAN 40 — Corporate      VLAN 50 — PKI
              Applications / Data         CA01
                    |
          WEB03 / Gitea / SQL-related
               internal services
```

This diagram represents the functional separation of the laboratory. It is not a complete physical cabling or interface-level diagram, and it does not imply that every VLAN permits direct communication with every other VLAN.

Actual connectivity is determined by the configured interfaces, routes and firewall rules.

---

## Network Segments

The laboratory uses the following functional VLAN assignments.

| Segment | Network / Role                   | Known systems                                                |
| ------- | -------------------------------- | -------------------------------------------------------------|
| VLAN 10 | Root Active Directory            | DC01                                                         |
| VLAN 20 | Corporate Active Directory       | DC02, DC03                                                   |
| VLAN 30 | Web infrastructure DMZ           | WEB01, WEB02                                                 |
| VLAN 40 | Corporate Applications and Data  | Gitea, WEB03, mail,database-related systems and applications |
| VLAN 50 | PKI infrastructure               | CA01                                                         |
| VLAN 60 | Workstations and internal access | WK01, WK03, WK04                                             |

The VLAN identifiers describe the logical segmentation. Exact subnet assignments should be confirmed against the current VirtualBox and pfSense configuration before being treated as authoritative.

### VLAN 10 — Root Active Directory

This segment contains the root-domain controller:

```text
DC01
192.168.10.10
Domain: apexfiend.lab
```

The segment represents the forest-root infrastructure and provides a separate network zone for root-domain services.

### VLAN 20 — Corporate Active Directory

This segment contains the domain controllers for the child domain:

```text
DC02
172.16.20.100

DC03
172.16.20.101

Domain: corp.apexfiend.lab
```

DC02 provides PDC and DNS functionality. DC03 provides Global Catalog and DNS functionality.

The segment supports corporate-domain authentication, directory services and DNS resolution.

### VLAN 30 — Web Infrastructure

VLAN 30 hosts the externally exposed or DMZ-oriented web infrastructure.

Known systems:

| Hostname | Domain membership | Role |
|---|---|---|
| WEB01 | Not domain-joined | DMZ web server |
| WEB02 | Not domain-joined | DMZ web server |

These systems are separated from the corporate Active Directory environment.

The DMZ represents a distinct security zone in which compromise of a web server should not automatically provide access to internal domain resources.

Any permitted communication between the DMZ and internal services must be documented in the firewall configuration.

### VLAN 40 — Corporate Applications and Data

VLAN 40 hosts internal application and data services within the corporate network.

Known systems and services include:

- WEB03
- Gitea
- mail01
- SQL-related infrastructure and application dependencies

WEB03 hosts Mattermost and osTicket and is part of the corporate environment.

Unlike WEB01 and WEB02, WEB03 is associated with the internal domain network.

The segment represents the application layer that connects corporate users and services with internal data and infrastructure.

### VLAN 50 — PKI Infrastructure

This segment contains the Enterprise Certificate Authority:

```text
CA01
172.16.50.100

CA: corp-CA01-CA
```

The separation of PKI infrastructure provides a distinct security boundary for certificate services.

Access to the CA and related management services should be restricted according to operational requirements.

### VLAN 60 — Workstations and Internal Access

VLAN 60 contains five Windows workstations used for different corporate roles.

| Hostname | IP address | Role |
|---|---|---|
| WK01 | `172.16.60.101` | Standard corporate workstation |
| WK02 | `172.16.60.102` | Web Developer / DevOps Engineer |
| WK03 | `172.16.60.103` | PKI operations and certificate inventory |
| WK04 | `172.16.60.104` | Jenkins maintenance and automation |
| WK05 | `172.16.60.105` | Help Desk L1 workstation |

The VPN network is `10.0.200.0/24`. VPN clients use this network to obtain controlled access to the internal environment according to the configured routing and firewall policies.
```

---

## External and VPN Access

The simulated development network uses the following known addresses:

```text
Development workstation:
10.0.100.11

VPN-related address:
10.0.100.12

pfSense WAN:
10.0.100.6
```

The VPN provides an entry point into the internal workstation environment.

The precise interface assignment and routing behavior should be verified against the current VPN and pfSense configuration. The addresses above should not be interpreted as proof that all three systems share the same interface or routing role.

The intended access model separates the initial development context from the internal infrastructure and uses network controls to determine which internal services are reachable.

---

## Routing and Connectivity

pfSense acts as the central routing and firewall component.

Its responsibilities include:

* Routing traffic between configured network segments
* Enforcing inter-VLAN firewall rules
* Providing VPN-based access
* Restricting access to sensitive infrastructure
* Supporting controlled connectivity between applications and their dependencies

The presence of a route between two networks does not necessarily mean that traffic is permitted between them.

The effective connectivity model depends on both routing and firewall policy.

Specific permitted flows, blocked connections and associated validation results are documented in `firewall-and-routing.md`.

---

## Active Directory Network Relationships

The Active Directory forest contains:

```text
apexfiend.lab
└── corp.apexfiend.lab
```

The root and corporate domains occupy separate network segments.

This separation makes it possible to examine the interaction between network reachability and directory-service relationships.

Network segmentation and Active Directory trust are separate concepts: a domain trust does not itself guarantee network connectivity, and a firewall restriction does not remove an existing directory trust.

The exact domain trust configuration and relevant directory permissions are documented under `02-active-directory/`.

---

## Application and Data Flows

The laboratory includes dependencies between web applications, databases, collaboration services and administrative infrastructure.

The primary systems of interest include:

* WEB01, WEB02 and WEB03
* Gitea
* MSSQL01 and MSSQL02
* MAIL01
* CA01
* Jenkins infrastructure
* Windows workstations

These dependencies are relevant to internal security testing because application access, service authentication and network reachability can combine to create paths between otherwise separate systems.

Not every potential communication path is assumed to be permitted. Each dependency must be checked against the actual service configuration and firewall policy.

Detailed service relationships are documented in `infrastructure-dependencies.md`.

---

## Security Design Considerations

The network architecture is intended to support the evaluation of the following controls:

### Segmentation

Different infrastructure roles are placed in separate logical network segments rather than sharing a single flat network.

### Restricted access to sensitive systems

Domain controllers, PKI infrastructure and database servers should be reachable only from the systems and services that require access.

### Controlled initial access

VPN connectivity provides a defined entry point into the internal environment.

### Application-to-database restrictions

Application servers should have only the database connectivity required for their functions.

### Administrative boundaries

Administrative workstations and infrastructure management services should not automatically be reachable from every internal segment.

These are architectural objectives, not claims that every corresponding firewall rule has already been implemented and validated.

---

## Validation

Network validation should be performed progressively as the laboratory evolves.

Relevant checks include:

* Confirming interface assignments and subnet masks
* Verifying IP addressing and default gateways
* Testing DNS resolution
* Checking domain-controller connectivity
* Testing required application-to-database connections
* Confirming VPN routing
* Verifying permitted and blocked inter-VLAN traffic
* Confirming that external access is restricted as intended

Validation results should record the source, destination, protocol, expected behavior and observed result.

This distinction helps separate the intended network design from the effective configuration.

---

## Related Documentation

* `lab-setup.md` — General laboratory setup
* `system-inventory.md` — Known systems and their roles
* `firewall-and-routing.md` — Routing and firewall behavior
* `infrastructure-dependencies.md` — Dependencies between services
* `trust-boundaries.md` — Security boundaries and their implications
* `02-active-directory/` — Active Directory architecture and configuration
