# Apexfiend — WEB03

## 1. Purpose

This document records the verified configuration of `WEB03`, the domain-joined Ubuntu host that consolidates the lab's internal collaboration and support applications — Mattermost and osTicket — behind a single nginx reverse proxy, alongside a landing portal that links out to both.

```text
Hostname:     web03
IP address:   172.16.40.103   (VLAN 40 — Corporate Applications and Data)
Domain:       corp.apexfiend.lab (domain-joined via SSSD — confirmed, see Section 6)
```

## 2. Reverse Proxy (nginx)

```powershell
grep -RiE 'listen|server_name|proxy_pass|root' /etc/nginx/sites-enabled/*
```

nginx listens on 80/443 and proxies three hostnames to three distinct local backends:

| Hostname | Proxies to | Backend |
|---|---|---|
| `portal.apexfiend.lab` | `127.0.0.1:8080` | Apache2 — static landing page |
| `helpdesk.apexfiend.lab` | `127.0.0.1:8081` | Apache2 — osTicket |
| `chat.apexfiend.lab` | `127.0.0.1:8065` | Mattermost (native binary) |

All three hostnames are served from this single nginx instance on the same host. **`portal.apexfiend.lab` is a landing page that links out to `chat.apexfiend.lab` and `helpdesk.apexfiend.lab`** — functionally the equivalent, for the internal corporate network, of what `WEB01`'s public landing page does for the DMZ.

## 3. Apache2 Virtual Hosts

```powershell
cat /etc/apache2/sites-available/portal.conf
cat /etc/apache2/sites-available/helpdesk.apexfiend.lab.conf
```

```apache
<VirtualHost 127.0.0.1:8080>
    ServerName portal.apexfiend.lab
    DocumentRoot /var/www/html
</VirtualHost>

<VirtualHost 127.0.0.1:8081>
    ServerName helpdesk.apexfiend.lab
    DocumentRoot /var/www/osticket/
    <Directory /var/www/osticket/>
        AllowOverride All
        Require all granted
    </Directory>
</VirtualHost>
```

## 4. Mattermost

```powershell
grep -E '"SiteURL"|"ListenAddress"|"DriverName"|"EnableOpenServer"|"RestrictCreationToDomains"' /opt/mattermost/config/config.json
```

```text
SiteURL:                   https://chat.apexfiend.lab
ListenAddress:              127.0.0.1:8065
EnableOpenServer:            true
RestrictCreationToDomains:   ""   (empty)
DriverName (SqlSettings):     postgres
DriverName (FileSettings):    local
ListenAddress (metrics):      :8067
```

Runs as the native Mattermost binary (`/opt/mattermost/bin/mattermost`, systemd-managed), backed by local PostgreSQL 16 (visible in `ps aux` as `postgres: 16/main: mmuser mattermost ...`). File storage uses the local driver (no S3/external object storage). Three plugins are loaded: `mattermost-ai`, `playbooks`, and `com.mattermost.calls`.

**Verified finding, needs confirmation against design intent:** `EnableOpenServer: true` combined with an empty `RestrictCreationToDomains` means **self-registration is fully open to anyone who can reach `chat.apexfiend.lab`, with no email-domain restriction at all.** The lab's documented design (`00-overview/lab-setup.md`) describes registration as gated through an osTicket support-ticket/email-verification step — if that gating is meant to be enforced, it is not being enforced by Mattermost's own configuration; nothing here would stop direct self-registration bypassing osTicket entirely. This should be confirmed as intentional (a deliberately weak control the student is meant to notice) or tightened.

## 5. osTicket

Served by Apache2/PHP at `/var/www/osticket/` (Section 3). PHP is handled via PHP-FPM 8.3 (`php-fpm: pool www`, confirmed in `ps aux`). Department/queue structure has not yet been captured — pending access to the osTicket admin panel.

A MariaDB instance (`mysql` user, `ps aux`) runs on this host; osTicket's standard backend is MySQL/MariaDB, so this is presumed to be its database, though not independently confirmed against osTicket's own configuration file (not queried, to avoid capturing its database credentials).

## 6. Domain Integration

```text
/usr/libexec/sssd/sssd_be --domain corp.apexfiend.lab
/usr/sbin/sssd -i --logger=files
sssd_nss, sssd_pam, sssd_pac
```

**Confirms `WEB03` is domain-joined to `corp.apexfiend.lab` via SSSD** (System Security Services Daemon) — the standard mechanism for Linux hosts to authenticate against Active Directory. This matches the domain-membership status already recorded in `01-architecture/network-architecture.md` and `infrastructure-dependencies.md`. The presence of `sssd_pac` specifically indicates Kerberos PAC (Privilege Attribute Certificate) processing is active, meaning domain users' group memberships are available locally on this host for authorization decisions (e.g., SSH access, `sudo` rules) — relevant to any attack-path step that assumes a domain identity can log into `WEB03` directly.

## 7. Open Items — Need Clarification

Two things turned up in `ps aux` and `ss -tuln` that are not yet accounted for in the lab's documented design:

1. **Postfix and Dovecot are running on `WEB03` itself** (`/usr/lib/postfix/sbin/master`, `/usr/sbin/dovecot`, with IMAP/IMAPS listening on `143`/`993`). This is unexpected — `MAIL01` was documented as the dedicated corporate mail server. Is this a local mail stack with its own purpose on `WEB03` (e.g., piping osTicket ticket notifications, or local-only system mail), or does it indicate `WEB03` plays a larger role in mail handling than currently documented?
2. **An unidentified service on port 8443** (both TCP on `0.0.0.0:8443`/`127.0.0.1:8443` and UDP on `172.16.40.103:8443`) does not map to any process confirmed in the `ps aux` capture. What is this?

Both are left unresolved in this document pending clarification, rather than guessed at.

## 8. Listening Ports Summary

```powershell
ss -tuln
```

| Port | Bind | Service |
|---|---|---|
| 22 | `0.0.0.0` / `[::]` | SSH |
| 53 | `127.0.0.53`, `127.0.0.54` | Local DNS stub resolver(s) |
| 80, 443 | `0.0.0.0` | nginx (reverse proxy) |
| 143, 993 | `0.0.0.0` / `[::]` | Dovecot (IMAP/IMAPS) — see Section 7 |
| 8065 | `127.0.0.1` | Mattermost |
| 8080 | `127.0.0.1` | Apache2 — portal |
| 8081 | `127.0.0.1` | Apache2 — osTicket |
| 8443 | `0.0.0.0`, `127.0.0.1` (TCP+UDP) | Unidentified — see Section 7 |

## 9. Validation Status

| Item | Status |
|---|---|
| nginx reverse-proxy mapping | Verified |
| Apache2 virtual hosts | Verified |
| Mattermost configuration (SiteURL, listen address, registration settings) | Verified |
| Domain integration via SSSD | Verified |
| osTicket backend database (MariaDB) | Presumed, not independently confirmed |
| Purpose of Postfix/Dovecot on this host | Not yet clarified |
| Service on port 8443 | Not yet identified |
| osTicket department/queue structure | Not yet captured |

## 10. Related Documentation

- `osticket.md` *(pending)* — Department structure and ticket workflow detail
- `mattermost.md` *(pending)* — Channel structure detail
- `03-network-services/dns.md` — Confirms `web03` at `172.16.40.103`
- `01-architecture/infrastructure-dependencies.md` — Domain-membership status this document verifies