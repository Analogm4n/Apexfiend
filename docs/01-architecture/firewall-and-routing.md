# Apexfiend — Firewall and Routing

## 1. Purpose

This document describes the firewall and routing architecture of the Apexfiend lab. It outlines the role of pfSense, the network segments, the VPN access path, and the intended security boundaries between infrastructure components.

Apexfiend uses network segmentation to represent an enterprise environment in which workstations, domain controllers, application servers, database systems, and PKI infrastructure do not all share the same network.

This document describes the architecture and intended security model. Individual firewall rules and their enforcement status should be documented separately once verified against the active pfSense configuration.

## 2. Firewall Platform

**Platform:** pfSense
**Role:** Network gateway, inter-network routing, and firewall enforcement
**WAN address:** `10.0.100.6`

pfSense connects the lab's network segments and provides the control point for traffic between networks, subject to its configured interfaces, routes, and firewall rules.

The firewall is intended to support the following functions:

- Route traffic between configured internal networks when permitted.
- Enforce access restrictions between network segments.
- Provide the external-facing connectivity used by the lab's development workstation.
- Support remote access through the lab VPN.
- Restrict access to sensitive infrastructure according to the intended trust boundaries.

The presence of a network route does not imply that traffic is permitted. Actual connectivity depends on the firewall rules and service-level controls in place.

## 3. Network Segmentation

The lab is divided into logical network segments. The following table summarizes their intended roles.

| Segment | Network / VLAN | Purpose |
|---|---|---|
| Root Active Directory | VLAN 10 | Root-domain domain controller and related services |
| Corporate Active Directory | VLAN 20 | Corporate-domain domain controllers and DNS |
| DMZ | VLAN 30 | WEB01 and WEB02; external-facing or isolated web services |
| Corporate applications and data | VLAN 40 | WEB03, Gitea, and database infrastructure |
| PKI | VLAN 50 | CA01 and certificate services |
| Workstations | VLAN 60 | User workstations and administrative or operational endpoints |
| VPN clients | `10.0.200.0/24` | Remote access into the lab, subject to VPN and firewall configuration |

The exact interface addresses, subnet masks, and DHCP settings for individual segments should be recorded in the network configuration or inventory as they are confirmed.

## 4. DMZ and Corporate Application Network

### 4.1 DMZ — VLAN 30

WEB01 and WEB02 reside in VLAN 30 and are **not joined to Active Directory**.

This separation is intended to represent a boundary between web-facing systems and the internal corporate environment. A compromise of a DMZ host should not automatically provide unrestricted access to corporate systems or domain resources.

Traffic from the DMZ to other segments should be limited to explicitly required destinations and services. The actual restrictions must be verified against the active firewall rules.

### 4.2 Corporate Applications and Data — VLAN 40

VLAN 40 contains corporate application and data services, including:

- WEB03, which belongs to the corporate domain network.
- Gitea, used for source-code and project documentation hosting.
- SQL Server infrastructure, including MSSQL01 and MSSQL02.

These systems support application workflows and internal services. Access between them should be based on required service dependencies rather than unrestricted connectivity.

WEB03 is distinct from WEB01 and WEB02: it is located in VLAN 40 and participates in the corporate domain environment.

## 5. Active Directory and PKI Networks

### 5.1 Active Directory

VLAN 10 hosts the root-domain domain controller, DC01. VLAN 20 hosts the corporate-domain domain controllers, DC02 and DC03.

Communication between these networks may be required for domain operations, name resolution, authentication, and forest-related services. The required connectivity depends on the configured Active Directory topology and service dependencies.

The existence of separate VLANs should not be interpreted as proof that all inter-domain traffic is blocked or that all required ports are permitted. Both conditions must be verified from the active configuration.

### 5.2 PKI

CA01 resides in VLAN 50 and provides enterprise certificate services.

PKI-related communication may include certificate enrollment, certificate retrieval, directory access, and administrative operations. Access should be limited to the clients and services that require it.

The intended design separates certificate infrastructure from ordinary workstation and application networks. The effective level of isolation depends on the firewall rules, CA configuration, and permissions assigned within Active Directory Certificate Services.

## 6. Workstations and Remote Access

### 6.1 Workstations — VLAN 60

The workstation segment contains five systems with different operational roles:

| Host | Address | Role |
|---|---|---|
| WK01 | `172.16.60.101` | Standard corporate workstation |
| WK02 | `172.16.60.102` | Web development and DevOps |
| WK03 | `172.16.60.103` | PKI operations and certificate inventory |
| WK04 | `172.16.60.104` | Jenkins maintenance and automation |
| WK05 | `172.16.60.105` | Help Desk L1 |

Although these systems share a network segment, their user privileges and responsibilities differ. Network placement alone does not define their effective permissions.

### 6.2 VPN Network

The VPN client network is `10.0.200.0/24`.

VPN access provides a remote entry point into the lab. The networks and services reachable through the VPN depend on the VPN configuration, routing, firewall rules, and the privileges of the connecting user.

The VPN should be treated as an access boundary, not as evidence that a connected client has unrestricted access to every internal segment.

The development workstation currently has the address `10.0.100.11` on the external-facing network. This address is separate from the VPN client subnet.

## 7. Routing and Traffic Control

Routing determines which networks can be reached through the configured gateways. Firewall policy determines which traffic is allowed to cross the relevant interfaces.

The following traffic categories should be reviewed when validating the lab:

- **Workstation to Active Directory:** DNS, authentication, and other domain operations required by the workstation's role.
- **Application to database:** Connections required by application components using MSSQL01 or MSSQL02.
- **Corporate network to PKI:** Certificate enrollment, retrieval, and administrative operations where applicable.
- **DMZ to corporate network:** Explicitly required application traffic only.
- **VPN to internal networks:** Access limited to the intended remote-access scope.
- **Management traffic:** Administrative access restricted to the appropriate systems and accounts.
- **Inter-domain traffic:** Connectivity required by the root and corporate Active Directory environments.

These categories describe validation targets, not a claim that a particular rule currently exists.

## 8. Firewall Validation

The following checks can be used to document and verify the implementation:

- [ ] Record each pfSense interface, VLAN, subnet, and gateway.
- [ ] Export or document the active firewall rules for each interface.
- [ ] Confirm which internal networks are reachable from the VPN.
- [ ] Verify whether DMZ hosts can initiate connections to corporate application and data networks.
- [ ] Test the required application-to-database connections.
- [ ] Verify required Active Directory and DNS communication between domain controllers and clients.
- [ ] Confirm which systems can access CA01 and certificate enrollment services.
- [ ] Review management access to servers and network infrastructure.
- [ ] Record observed results and any deviations from the intended policy.

Testing should be performed from authorized lab systems and documented with the source, destination, protocol, port, and observed result.

## 9. Security Considerations

Network segmentation reduces unnecessary connectivity but does not independently prevent lateral movement. Its effectiveness depends on firewall policy, host configuration, identity permissions, and the services exposed by each system.

Apexfiend is designed to explore how weaknesses across these layers can combine into broader compromise paths. Firewall restrictions should therefore be evaluated alongside Active Directory permissions, application trust relationships, credential exposure, and service configuration.

## 10. Scope and Limitations

This document records the network security architecture and its intended controls. It does not constitute a verified list of every active firewall rule, route, NAT mapping, or VPN access policy.

Unconfirmed interface addresses and server addresses should be added after checking the running configuration. Any statement about effective connectivity should be supported by configuration evidence or a repeatable connectivity test.
