# Apexfiend



A self-built enterprise Active Directory security lab designed to

simulate a segmented corporate environment and practice internal

penetration testing, Active Directory exploitation, AD CS, Kerberos,

SQL Server, Jenkins and lateral movement.



## Overview

Apexfiend is a 20-VM enterprise-style lab built with VirtualBox and

pfSense.



The environment includes:

- Multi-domain Active Directory forest

- Windows Server infrastructure

- Segmented VLANs

- AD CS / Enterprise PKI

- SQL Server

- Jenkins

- Gitea

- Mattermost

- osTicket

- Postfix / Dovecot

- Internal web applications

- Workstations and service accounts



## Objectives

The lab was designed to practice:

- Internal network enumeration

- Active Directory enumeration

- Kerberos attacks

- ACL abuse

- SQL Server attack paths

- AD CS abuse

- NTLM relay

- Jenkins abuse

- gMSA

- RBCD (Resource-based Constrained Delegation)

- Lateral movement

- Privilege escalation

- Attack-chain documentation



## Documentation

- [Architecture](docs/01-architecture/)

- [Active Directory](docs/02-active-directory/)

- [Databases](docs/05-databases/)

- [PKI / AD CS](docs/06-pki/)

- [Jenkins](docs/07-jenkins/)

- [Attack Paths](docs/08-attack-paths/)

- [Development Log](docs/10-development-log/)

## Disclaimer

Apexfiend is a private security research and training environment

created for educational purposes.

