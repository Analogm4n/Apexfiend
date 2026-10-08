# Apexfiend

An Active Directory lab designed and built from scratch to practice and document offensive security / red teaming techniques against a simulated corporate environment: `corp.apexfiend.lab`.

The project covers both the **infrastructure design** (network, AD, PKI, Jenkins CI/CD, databases, internal web services) and the **full execution of an intentionally designed attack chain**, from external initial access to full domain compromise (DCSync).

## Why this project

Built to practice and demonstrate, end-to-end:

- Design of realistic Active Directory architectures (OUs, delegations, ACLs, gMSA).
- Configuration of Active Directory Certificate Services (ADCS) with an intentionally misconfigured certificate template (ESC1).
- Exploitation of trust relationships between SQL Server instances (linked servers).
- Abuse of Resource-Based Constrained Delegation (RBCD) and Shadow Credentials.
- Technical documentation at the level of a real production environment, including reporting in a professional pentest report format.

## Repository structure

```
Apexfiend/
├── README.md
├── CHANGELOG.md
├── LICENSE
├── .gitignore
│
├── docs/
│   ├── 00-overview/              # What the project is, objectives, scope, how to set it up
│   ├── 01-architecture/          # Network topology, VLANs, AD forest, diagrams
│   ├── 02-active-directory/      # Domains, OUs, users/groups, ACLs, Kerberos
│   ├── 03-network-services/      # pfSense, DNS, VPN, segmentation
│   ├── 04-web-and-collaboration/ # Web servers, Gitea, Mattermost, osTicket, mail
│   ├── 05-databases/             # SQL architecture, MSSQL01/02, trust boundaries
│   ├── 06-pki/                   # ADCS, CA01, certificate templates
│   ├── 07-jenkins/               # CI/CD architecture, jobs, gMSA, build agent
│   ├── 08-attack-paths/          # Designed attack chain, step by step, by vector
│   ├── 09-reporting/             # Findings report in professional pentest format
│   └── 10-development-log/       # Design milestones and decisions for the lab
│
├── lab/                          # Diagrams, configs, scripts, and provisioning templates
├── writeups/                     # Actual execution of the attack chain (enumeration → post-exploitation)
└── evidence/                     # Screenshots and sanitized output from the run-through
```

> The difference between `docs/08-attack-paths/` and `writeups/`: the former documents **the design** of the attack chain (the intent behind each step); the latter documents **the actual execution** (commands, output, screenshots), as if it were the fieldwork of a pentester solving the lab for the first time.

## Quick index

| Section | Description |
|---|---|
| [Overview](docs/00-overview/project-overview.md) | What Apexfiend is and why it exists |
| [Network architecture](docs/01-architecture/network-topology.md) | Topology, VLANs, segmentation |
| [Active Directory](docs/02-active-directory/domains.md) | Forest design, OUs, delegations |
| [PKI / ADCS](docs/06-pki/adcs-architecture.md) | CA01 and certificate templates |
| [Jenkins / CI-CD](docs/07-jenkins/architecture.md) | Automation, gMSA, health checks |
| [Full attack chain](docs/08-attack-paths/full-attack-chain.md) | From initial access to DCSync |
| [Findings and remediation](docs/09-reporting/findings.md) | Client-facing findings report |

## Project status

Under active development. See [CHANGELOG.md](CHANGELOG.md) for the change history and [docs/10-development-log/milestones.md](docs/10-development-log/milestones.md) for design milestones.

## Disclaimer

This is an isolated lab environment, built exclusively for educational and personal practice purposes. No component, credential, IP address, or data shown in this repository corresponds to real systems or information.

## Author

Jean Pierre Miranda Torres — Junior Penetration Tester.
