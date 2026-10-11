# Apexfiend — WEB01

## 1. Purpose

This document records the verified configuration of `WEB01`, the lab's public-facing DMZ web server and the confirmed technical entry point of the entire Apexfiend attack path.

```text
Hostname:     web01
IP address:   172.16.30.100   (VLAN 30 — DMZ)
Domain:       Not domain-joined
Public role:  "Apex Brand Cloud" — a decoy commercial front hosting three public subdomains
```

## 2. SSH Configuration

```powershell
grep -Ei '^(Port|AllowAgentForwarding|AllowTcpForwarding|PermitRootLogin|PasswordAuthentication|PubkeyAuthentication|X11Forwarding|AllowUsers|AllowGroups|ClientAliveInterval)' /etc/ssh/sshd_config
```

```text
X11Forwarding yes
AllowAgentForwarding yes
AllowTcpForwarding yes
```

**`AllowAgentForwarding yes` is the configuration that enables the SSH-agent-forwarding pivot** central to the early attack path: a user who gains access to `WEB01` via SSH with a forwarded agent can use that same agent to authenticate onward to other hosts the legitimate key-holder (the `dev` user) has access to — without ever obtaining the private key itself.

External SSH access reaches `WEB01` exclusively through the single NAT rule documented in `03-network-services/pfsense.md`, Section 4 (source restricted to `10.0.100.12`, destination TCP 22).

## 3. Running Services

```powershell
ps aux
ss -tlnp | grep -E ':22 |:80 |:443 '
```

```text
LISTEN  0.0.0.0:22    sshd
LISTEN  0.0.0.0:80    nginx (master + worker)
LISTEN  0.0.0.0:443   nginx (master + worker)
```

nginx is the sole public-facing listener on ports 80/443 and reverse-proxies to several backends running on `localhost` at different ports. Confirmed backend services and their local ports:

| Port | Service | Purpose |
|---|---|---|
| `127.0.0.1:8080` | Apache2 (`apexbrand-ssl.conf`) | Public corporate decoy site (`apexfiend.lab`), `DocumentRoot /var/www/html/corporate` |
| `127.0.0.1:8081` | Docker container (via `docker-proxy`) | Backend for `webmail.apexfiend.lab` |
| `127.0.0.1:8082` | `php -S 127.0.0.1:8082 -t /opt/webmail-api` | `webmail-api` — creates mailbox users in the Postfix/Dovecot container (see Section 5) |
| `127.0.0.1:8083` | Docker container (via `docker-proxy`) | Backend for `portal.apexfiend.lab` |

Additional locally-running components, not externally reachable: a local `bind`/`named` DNS resolver (Section 4), two independent `mariadbd` instances (Section 6), and the standard Postfix/Dovecot/OpenDKIM/OpenDMARC/amavisd mail stack supporting the webmail service (Section 5).

## 4. Public Subdomains

The site's public landing page (`apexfiend.lab`, served from Apache2 on 8080) advertises three subdomains, each a distinct service:

### job.apexfiend.lab

Links through to the "Job Recruitment" application. **This is the same application documented on `WEB02`/`MSSQL01`** — `WEB01` serves only as the public landing page and link-out; the application itself is hosted elsewhere.

### webmail.apexfiend.lab

A disposable/temporary email service, intended for public internet users — specifically, to let a visitor register a temporary mailbox so they can complete verification flows on `portal.apexfiend.lab` (see below). This is an independent mail stack, entirely separate from the corporate `MAIL01` server, running inside containers on `WEB01` itself (Postfix, Dovecot, OpenDKIM, OpenDMARC, amavisd — see Section 5).

### portal.apexfiend.lab

A file-upload utility advertised as SVG syntax/W3C-compliance validation ("Validación de Vectores"). **This is the confirmed technical entry point of the entire lab** — the upload functionality is intentionally vulnerable to XXE (XML External Entity injection) via crafted SVG files, combined with the disposable email service above (presumably used to receive a verification step as part of the upload/registration flow).

**Note:** `attack-path.md` Stage 1 currently describes initial access only generically ("VPN access / basic internal network connectivity") and does not yet reflect this specific technical mechanism. This should be reconciled — see the open question raised alongside this document.

## 5. Internal DNS

```text
root 789  /usr/sbin/named -f -u bind
```

`WEB01` runs its own local `bind`/`named` instance, resolving only the three subdomains above (`job.`, `webmail.`, `portal.apexfiend.lab`). **This is a standalone DNS zone local to WEB01's public-facing role — it is unrelated to, and separate from, the AD-integrated `apexfiend.lab` zone hosted on DC01/DC02 documented in `03-network-services/dns.md`.** The two should never be conflated: the public zone exists only to resolve WEB01's own decoy subdomains for internet visitors; the corporate zone is the real Active Directory forest-root zone.

## 6. Disposable Mail Stack

Supports `webmail.apexfiend.lab`. Running components (from `ps aux`):

```text
postfix (master, pickup, qmgr)
dovecot (+ anvil, log, config)
opendkim
opendmarc
amavisd (master + children)
```

A dedicated API, `webmail-api` (`php -S 127.0.0.1:8082 -t /opt/webmail-api`), handles mailbox creation for this disposable mail service — it provisions new mailbox users in the Postfix/Dovecot container on demand, which is how the public-facing temporary-email workflow is implemented.

This stack is entirely independent of `MAIL01` (`03-network-services/mail.md`, when written) and should not be confused with the corporate mail environment.

## 7. Databases

Two independent `mariadbd` instances run on `WEB01`:

| Instance | Backs |
|---|---|
| mariadbd (PID 1281) | `portal.apexfiend.lab` (upload/validation portal) |
| mariadbd (PID 1321) | `webmail.apexfiend.lab` (disposable mail service) |

No database credentials are recorded in this document.

## 8. Public Site Content

The Apache2-served landing page (`apexfiend.lab`, port 8080) is a decoy commercial front — "Apex Brand Cloud" — presenting the three subdomains above as legitimate public services (disposable email, SVG validation, job postings) with no indication of the corporate infrastructure behind them. This is consistent with the lab's broader design philosophy (`00-overview/project-overview.md`, Section 3): information and access are not directly exposed, they are discovered.

## 9. Validation Status

| Item | Status |
|---|---|
| SSH configuration | Verified |
| Listening services and reverse-proxy mapping | Verified |
| Public subdomain purposes | Verified (operator-confirmed) |
| XXE vulnerability in `portal.apexfiend.lab`'s SVG upload | Verified (operator-confirmed); not yet independently re-tested by Claude |
| Local DNS zone scope | Verified — confirmed isolated from the corporate AD zone |
| Webmail mailbox-provisioning mechanism | Verified |
| Database credentials / schema detail | Not recorded (out of scope for public documentation) |

## 10. Related Documentation

- `03-network-services/pfsense.md` — The NAT rule providing external SSH access to this host
- `03-network-services/dns.md` — The separate, unrelated corporate AD DNS zone
- `02-active-directory/` through `../02-active-directory/` — Not applicable; `WEB01` is not domain-joined
- `web02.md` — The Job Recruitment application `job.apexfiend.lab` links to
- `00-overview/attack-path.md` — Stage 1 (initial access); pending reconciliation with the XXE entry mechanism documented here