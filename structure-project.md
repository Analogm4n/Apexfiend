Apexfiend/

│

├── README.md

├── CHANGELOG.md

├── LICENSE

├── .gitignore

│

├── docs/

│   ├── 00-overview/

│   │   ├── project-overview.md

│   │   ├── objectives.md

│   │   ├── scope.md

│   │   ├── lab-setup.md                  

│   │   └── attack-path.md                

│   │

│   ├── 01-architecture/

│   │   ├── network-topology.md

│   │   ├── vlan-design.md

│   │   ├── ad-forest.md

│   │   ├── infrastructure.md

│   │   └── diagrams/

│   │       ├── network-topology.png

│   │       ├── ad-topology.png

│   │       └── attack-chain.png

│   │

│   ├── 02-active-directory/

│   │   ├── domains.md

│   │   ├── organizational-units.md

│   │   ├── users-and-groups.md

│   │   ├── delegated-permissions.md

│   │   ├── kerberos.md

│   │   └── acl-design.md

│   │

│   ├── 03-network-services/

│   │   ├── pfsense.md

│   │   ├── dns.md

│   │   ├── vpn.md

│   │   └── segmentation.md

│   │

│   ├── 04-web-and-collaboration/

│   │   ├── web-servers.md

│   │   ├── gitea.md

│   │   ├── mattermost.md

│   │   ├── osticket.md

│   │   └── mail.md

│   │

│   ├── 05-databases/

│   │   ├── sql-architecture.md

│   │   ├── mssql01.md

│   │   ├── mssql02.md

│   │   ├── recruitmentdb.md

│   │   ├── reportingdb.md

│   │   └── sql-trust-boundaries.md

│   │

│   ├── 06-pki/

│   │   ├── adcs-architecture.md

│   │   ├── ca01.md                       # summary + link to the pki-documentation repo

│   │   ├── certificate-templates.md      # summary + link, avoids duplicating templates/overview.md

│   │   └── certificate-inventory.md      # (if no external repo; if there is one, remove)

│   │

│   ├── 07-jenkins/

│   │   ├── architecture.md

│   │   ├── jobs.md

│   │   ├── build-agent.md                # renamed from dispatcher.md

│   │   ├── gmsa.md

│   │   └── automation-worker.md

│   │

│   ├── 08-attack-paths/

│   │   ├── initial-access.md

│   │   ├── web-to-vpn.md

│   │   ├── sql-path.md

│   │   ├── pki-path.md

│   │   ├── jenkins-path.md

│   │   ├── rbcd-path.md

│   │   └── full-attack-chain.md          # single source of truth for the full chain

│   │

│   ├── 09-reporting/

│   │   ├── executive-summary.md          # NEW — executive summary in real report style

│   │   ├── findings.md

│   │   ├── remediation.md

│   │   └── lessons-learned.md

│   │

│   └── 10-development-log/

│       ├── milestones.md

│       └── decisions.md                  # changelog.md removed from here (see CHANGELOG.md at root)

│

├── lab/

│   ├── diagrams/

│   ├── configs/

│   ├── scripts/

│   │   ├── powershell/

│   │   ├── sql/

│   │   ├── ldap/

│   │   └── automation/

│   └── vm-templates/                     # renamed from templates/ (avoids confusion with cert templates)

│

├── writeups/                             # actual execution: commands, output, screenshots

│   ├── enumeration/

│   ├── exploitation/

│   ├── lateral-movement/

│   ├── privilege-escalation/

│   └── post-exploitation/

│

└── evidence/

&#x20;   ├── screenshots/

&#x20;   └── sanitized-output/

