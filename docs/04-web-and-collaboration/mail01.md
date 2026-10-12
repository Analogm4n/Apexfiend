# Apexfiend — MAIL01

## 1. Purpose

This document records the verified configuration of `MAIL01`, the corporate mail server for `corp.apexfiend.lab`, as retrieved from the host directly.

```text
Hostname:     mail01.apexfiend.lab
IP address:   172.16.40.106   (VLAN 40 — Corporate Applications and Data)
Stack:        Postfix (SMTP) + Dovecot (IMAP, LMTP delivery)
AD integration: Confirmed — authentication against Active Directory via LDAP bind (Section 4)
```

Mailbox addresses use the `user@corp.apexfiend.lab` format, authenticated directly against the user's AD `userPrincipalName`. The server's own hostname remains under the root zone (`mail01.apexfiend.lab`) — see Section 5 for how this interacts with mail routing.

## 2. Postfix Configuration

```bash
postconf -n | grep -E '^(myhostname|mydomain|mydestination|relayhost|inet_interfaces|smtpd_sasl_auth_enable|smtpd_sasl_type|smtpd_sasl_path|virtual_alias_domains|virtual_mailbox_domains|virtual_transport)'
```

```ini
inet_interfaces = all
myhostname = mail01.apexfiend.lab
mydestination = $myhostname, mail01.apexfiend.lab, localhost.apexfiend.lab, localhost
relayhost =
smtpd_sasl_auth_enable = yes
smtpd_sasl_type = dovecot
smtpd_sasl_path = private/auth
virtual_mailbox_domains = corp.apexfiend.lab
virtual_transport = lmtp:unix:private/dovecot-lmtp
```

Postfix accepts mail for the `corp.apexfiend.lab` virtual mailbox domain and hands delivery off to Dovecot via LMTP — the standard Postfix+Dovecot integration pattern. SMTP client authentication is delegated to Dovecot's SASL mechanism (`smtpd_sasl_type = dovecot`), which in turn authenticates against AD (Section 4). No `relayhost` is configured — outbound mail is sent directly rather than relayed through an upstream smarthost.

## 3. Dovecot Configuration

```bash
doveconf -n | grep -E '^(mail_location|auth_mechanisms|ssl|listen|passdb|userdb|ldap)'
```

```ini
mail_location = maildir:/var/mail/vhosts/%d/%n
ssl = required
ssl_cert = </etc/ssl/private/mail01.cer
```

Mailboxes are stored as Maildir, one directory per domain/user (`%d/%n`). TLS is required for all connections; the certificate path confirms a dedicated certificate for this host (not reproduced here, and its issuing CA has not been separately verified against `06-pki/`).

## 4. LDAP Authentication (AD Integration)

```bash
grep -E '^(hosts|ldap_version|base|scope|user_filter|pass_filter|auth_bind)' /etc/dovecot/dovecot-ldap.conf.ext
```

```ini
hosts = 172.16.20.101
base = DC=corp,DC=apexfiend,DC=lab
ldap_version = 3
auth_bind = yes
auth_bind_userdn = %u
user_filter = (&(objectClass=user)(userPrincipalName=%u))
pass_filter = (&(objectClass=user)(userPrincipalName=%u))
```

Dovecot authenticates directly against `DC03` (`172.16.20.101`) using an LDAP simple bind (`auth_bind = yes`), matching the user's full `userPrincipalName` (i.e., `user@corp.apexfiend.lab`) rather than a bind-then-search pattern with a service account. This confirms and matches the UPN-based LDAP authentication already noted as tested in `01-architecture/infrastructure-dependencies.md`, Section 8. No separate mail-service bind account is used — authentication succeeds or fails based entirely on the mail client's own AD credentials.

## 5. DNS / Mail Routing

```bash
dig apexfiend.lab MX +short
dig corp.apexfiend.lab MX +short
```

```text
apexfiend.lab      MX  10 mail01.apexfiend.lab.
corp.apexfiend.lab  MX  (no record)
```

**Needs clarification, not treated as an error:** the published MX record is for the root zone (`apexfiend.lab`), while the mailbox domain Postfix actually accepts mail for is the child zone (`virtual_mailbox_domains = corp.apexfiend.lab`), which has no MX record of its own. Mail addressed to `user@corp.apexfiend.lab` therefore has no DNS-based routing path distinct from internal delivery — any host that needs to reach `MAIL01` for `corp.apexfiend.lab` mail presumably relies on direct internal configuration (static relay, internal resolution, or simply being handed `mail01.apexfiend.lab` directly) rather than an MX lookup on the mailbox domain itself. This is worth confirming as intentional before being treated as a finding.

## 6. Running Services

```bash
ps aux | grep -E 'postfix|dovecot|opendkim|opendmarc|amavis'
```

```text
dovecot (+ anvil, log, config)
postfix (master, pickup, qmgr, tlsmgr)
```

**No OpenDKIM, OpenDMARC, or amavis processes are running on `MAIL01`.** This is a meaningful difference from the disposable mail stack on `WEB01` (`web01.md`, Section 5), which does run all three — `MAIL01`, the actual corporate mail server, has no DKIM signing, no DMARC verification, and no content filtering/antivirus scanning active.

## 7. Validation Status

| Item | Status |
|---|---|
| Postfix configuration | Verified |
| Dovecot configuration | Verified |
| LDAP/AD authentication integration | Verified |
| MX record for `apexfiend.lab` | Verified |
| Absence of an MX record for `corp.apexfiend.lab` | Verified; routing implication not yet confirmed as intentional |
| Running service set (no DKIM/DMARC/content filtering) | Verified |

## 8. Related Documentation

- `web01.md` — Contrast with the separate disposable mail stack on `WEB01`, which does run DKIM/DMARC/amavis
- `03-network-services/dns.md` — `MAIL01`'s A record placement in the root zone
- `01-architecture/infrastructure-dependencies.md` — Section 8 already documented this LDAP authentication path at a design level; this document confirms it against the live configuration
- `02-active-directory/domains.md` — `DC03` (`172.16.20.101`), the LDAP bind target