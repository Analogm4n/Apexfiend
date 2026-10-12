# Apexfiend — Gitea

## 1. Purpose

This document records the verified configuration of the lab's internal Gitea instance, hosted on `GITEA01`.

```text
Hostname:     gitea01
IP address:   172.16.40.100   (VLAN 40 — Corporate Applications and Data)
Domain:       Domain-joined (host-level) — Gitea's own authentication is local, not AD-integrated
Public URL:   https://gitea.apexfiend.lab/
```

Secrets present in the live configuration (`INTERNAL_TOKEN`, `SECRET_KEY`, `LFS_JWT_SECRET`, OAuth2 `JWT_SECRET`, and any API tokens used to query the instance) have been deliberately excluded from this document.

## 2. Deployment

```yaml
# docker-compose.yml
services:
  gitea:
    image: gitea/gitea:latest
    environment:
      - GITEA__database__DB_TYPE=sqlite3
      - GITEA__server__ROOT_URL=https://gitea.apexfiend.lab/
      - GITEA__server__SSH_DOMAIN=gitea.apexfiend.lab
      - GITEA__server__SSH_PORT=2222
      - GITEA__server__DOMAIN=gitea.apexfiend.lab
    ports:
      - '127.0.0.1:3000:3000'
      - '2222:22'
```

```powershell
docker ps --filter "name=gitea"
```

```text
Image:   gitea/gitea:latest
Ports:   127.0.0.1:3000 -> 3000/tcp (web, local-only)
         0.0.0.0:2222 -> 22/tcp (Git SSH, externally reachable)
Status:  Up
```

The web UI is bound to `localhost` only and reached exclusively through the nginx reverse proxy (Section 3). Git-over-SSH, however, is bound to all interfaces on port 2222 and is directly reachable from the network without going through nginx.

## 3. Reverse Proxy

```nginx
server {
    listen 80;
    server_name gitea.apexfiend.lab;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name gitea.apexfiend.lab;
    ssl_certificate     /etc/nginx/ssl/gitea.apexfiend.lab.cer;
    ssl_certificate_key /etc/nginx/ssl/gitea.apexfiend.lab.key;
    location / {
        proxy_pass http://127.0.0.1:3000;
        client_max_body_size 512M;
    }
}
```

HTTP is redirected to HTTPS unconditionally. The TLS certificate is a dedicated `gitea.apexfiend.lab` certificate (not the AD CS-issued enterprise CA chain documented in `06-pki/`) — its issuance path has not been separately investigated. `client_max_body_size 512M` accommodates Git LFS uploads (`LFS_START_SERVER = true` in `app.ini`).

## 4. Application Configuration

Non-sensitive sections of `app.ini`:

```ini
[server]
DOMAIN = gitea.apexfiend.lab
HTTP_PORT = 3000
SSH_PORT = 2222
SSH_LISTEN_PORT = 22
LFS_START_SERVER = true

[database]
DB_TYPE = sqlite3
PATH = /data/gitea/gitea.db

[security]
PASSWORD_HASH_ALGO = pbkdf2

[service]
DISABLE_REGISTRATION = false
REQUIRE_SIGNIN_VIEW = false
REGISTER_EMAIL_CONFIRM = false
ALLOW_ONLY_EXTERNAL_REGISTRATION = false
ENABLE_CAPTCHA = false
DEFAULT_ALLOW_CREATE_ORGANIZATION = true

[mailer]
ENABLED = false

[openid]
ENABLE_OPENID_SIGNIN = true
ENABLE_OPENID_SIGNUP = true
```

The database backend is **SQLite**, not a dedicated MySQL/MariaDB/PostgreSQL instance — a basic, single-file installation rather than a clustered or externally-managed database.

**Verified finding:** self-registration is fully open (`DISABLE_REGISTRATION = false`), with no email confirmation (`REGISTER_EMAIL_CONFIRM = false` — and `[mailer] ENABLED = false` means confirmation mail could not be sent even if required), no CAPTCHA, and OpenID self-signup also enabled. Combined with `REQUIRE_SIGNIN_VIEW = false`, public repositories are browsable by anyone without any account at all, and creating an account to interact further requires no verification step whatsoever. This mirrors the open-registration finding already documented for Mattermost in `web03.md`, Section 4 — the pattern of weak registration controls appears more than once across the lab's collaboration tooling.

**Confirmed by the operator:** Gitea's own authentication is local (username/password via `pbkdf2`), not integrated with Active Directory, even though the host `GITEA01` itself is domain-joined. A Gitea account does not correspond to, or require, a domain identity.

## 5. Repository Inventory

```text
curl -H "Authorization: token <redacted>" https://gitea.apexfiend.lab/api/v1/repos/search?limit=50
```

| Repository | Owner | Visibility | Role |
|---|---|---|---|
| `legacy-server-setup` | `IT-Help-Desk` | Private | Presumed: the historical installation-documentation repository carrying the old shared local Administrator password (`00-overview/attack-path.md` early-game design) — pending confirmation |
| `pki-documentation` | `PKI-Repository` | Private | Confirmed: the PKI documentation repository already fully drafted elsewhere in this project |
| `developer-deployment` | `dev` | Private | Not yet documented — purpose pending |
| `infrastructure-automation` | `m.ortega` | Private | Not yet documented — purpose pending |
| `internal-dashboard` | `dev` | **Public** | Not yet documented — purpose pending; notable for being readable without authentication (see Section 4 finding) |
| `jenkins-operations` | `m.alvarez` | Private | Not yet documented — purpose pending |
| `mssql-documentation` | `d.garcia` | Private | Not yet documented — purpose pending |
| `web-services` | `Web-Operations` (org) | Private | Not yet documented — purpose pending |

Repository ownership by individual users (`dev`, `m.ortega`, `m.alvarez`, `d.garcia`) and by organizations (`IT-Help-Desk`, `PKI-Repository`, `Web-Operations`) mirrors the role structure already documented in `02-active-directory/users-and-groups.md`, even though Gitea accounts are entirely separate from the corresponding AD identities.

## 6. Validation Status

| Item | Status |
|---|---|
| Docker deployment and port mapping | Verified |
| nginx reverse proxy configuration | Verified |
| Application configuration (server, database, service, security) | Verified |
| AD integration status | Verified — not integrated, confirmed by operator |
| Registration/sign-in security posture | Verified |
| Repository list (names, owners, visibility) | Verified |
| Repository contents and narrative role (6 of 8) | Not yet documented — pending clarification |

## 7. Related Documentation

- `web03.md` — Shares the open-registration finding pattern (Mattermost)
- `03-network-services/dns.md` — Confirms `GITEA01` at `172.16.40.100`
- `pki-documentation/` *(separate repository)* — Full content of the `pki-documentation` repo referenced here