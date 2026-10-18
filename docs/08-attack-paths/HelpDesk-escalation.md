# Apexfiend — Help Desk Escalation (d.ramirez → m.rodriguez → d.garcia)

## 1. Purpose

This document continues from `web-to-vpn.md`: an authenticated RDP session as `corp\d.ramirez` on `WK01`. It narrates the confirmed, tested chain from there through to code execution as `corp\d.garcia` on `WK02` — the real-world execution of the Help Desk escalation chain already analyzed at the ACL level in `02-active-directory/acl-design.md`, Section 3.

## 2. d.ramirez — Enumeration

```text
Full name:    Diego Ramirez
Comment:      Web Developer / DevOps Engineer WK01
Group:        GG-WebOperations, Domain Users
```

`GG-WebOperations`'s membership is confirmed as exactly two users: `d.ramirez` and `d.garcia` — matching `02-active-directory/users-and-groups.md`. Outlook (already installed) is accessed using `d.ramirez`'s mail credentials, recovered from the KeePass vault on `DEV-LAPTOP` (`web-to-vpn.md`, Section 3).

## 3. Mail and Gitea — Confirming the Application Landscape

Email correspondence between `d.ramirez` and `d.garcia` confirms, in their own words, that the `JobPortal` application is hosted on `WEB02` with its data on `MSSQL01` — independently corroborating `04-web-and-collaboration/web02.md` and `05-databases/jobportal.md`. A separate onboarding email grants `d.ramirez` access to the `Web-Operations/web-services` Gitea repository (the sixth previously-undocumented repository flagged in `gitea.md`, now resolved as the team's own deployment/documentation repo).

**Gitea login behavior confirms `gitea.md`, Section 4:** `d.ramirez`'s domain password does **not** work against Gitea (consistent with Gitea's authentication being local, not AD-integrated). A personal password recovered from the Firefox credential harvest on `DEV-LAPTOP` (`web-to-vpn.md`, Section 3) — reused by `d.ramirez` for his Gitea account — succeeds instead.

Inside `web-services`, documentation covers `WEB01`, `WEB02`, `WEB03`, `MSSQL01`, and `MSSQL02` (the MSSQL02 reference predates the MSSQL02→MSSQL03 migration, consistent with the stale naming already noted in `07-jenkins/jobs.md`). The `web03.md` document in this repo names three endpoints on `WEB03`: `helpdesk.apexfiend.lab` (osTicket), `chat.apexfiend.lab` (Mattermost), and `portal.apexfiend.lab`.

**Naming note:** `portal.apexfiend.lab` resolves differently depending on which DNS server a client uses — `WEB01`'s own isolated, internet-facing DNS zone (`web01.md`, Section 5) resolves it to the XXE-vulnerable upload portal on `WEB01` itself; the corporate AD-integrated zone (`03-network-services/dns.md`) resolves the same name to `WEB03`'s landing page. Both are real, intentional, and simply reachable from different vantage points — not a documentation error.

## 4. osTicket → Mattermost Registration

`helpdesk.apexfiend.lab` (osTicket) accepts ticket creation from any email address, including external ones. A test ticket, created with an external address, returns both a ticket number and an email address for follow-up correspondence in the form `<ticket-number>@tickets.apexfiend.lab`. **This ticket-specific email address is then used to self-register a Mattermost account** — satisfying whatever email-based gate exists on account creation without needing a real corporate mailbox. This confirms the registration path originally anticipated at a design level (`00-overview/lab-setup.md`), operating alongside the independently-documented fact that Mattermost's own server configuration has no actual domain restriction or email verification enforced (`web03.md`, Section 4) — the ticket-email route works, but a more direct registration may well have worked too.

## 5. Mattermost — Discovering the Onboarding Pattern

Once inside Mattermost, the `#welcome` and `#it-onboarding` channels reference several recently-onboarded Help Desk hires, including **Laura Sanchez**. A message from Laura specifically asks for help because her VPN password — tied by configuration to her domain password — isn't working, strongly implying she has not yet changed her temporary onboarding password.

Separately, **`d.ramirez`'s own historical welcome email**, found by searching further back in his already-accessed Outlook mailbox, reveals the organization's new-hire password convention:

```text
Username pattern: <initial><surname>
Password pattern: <initial><Surname><DeptCode><year>!   (e.g., d.ramirez → DRamirezWOP26!)
```

Applying the same pattern to Laura Sanchez, Help Desk, yields a predicted password — and it works. **This is a real, successfully-exploited password-pattern weakness**, not an assumption: the organization's own onboarding process is the vulnerability.

## 6. As Laura — Reaching Miguel's Dormant Account

Authenticated as `l.sanchez`, her delegated rights are re-confirmed directly against the live AD, matching `02-active-directory/delegated-permissions.md` exactly: `GG-HelpDesk-L1` holds `ReadProperty`/`WriteProperty` (including the `Account-Restrictions`-equivalent attributes) and the Reset Password `ExtendedRight` on both `OU=HelpDesk-L1` (self) and `OU=HelpDesk-L2`.

`m.rodriguez` (Miguel) is confirmed **disabled** (`Account active: No`), a member of `GG-HelpDesk-L2` and `GG-WebOperations-Support`. Using Laura's delegated rights over `OU=HelpDesk-L2`, his account is re-enabled and given a new password:

```powershell
Enable-ADAccount -Identity "m.rodriguez"
net user m.rodriguez <new-password> /domain
```

This is the deliberately "loud" option — a password reset — applied to the one target where it carries no real-world cost, exactly as analyzed in `acl-design.md`, Section 3.2: Miguel's account was never in active use, so no legitimate user is inconvenienced or alerted.

## 7. As Miguel — Shadow Credentials Against Daniel

`m.rodriguez`'s membership in `GG-WebOperations-Support` is confirmed to carry `WriteProperty` on `msDS-KeyCredentialLink` over `OU=WebOperations` — matching `delegated-permissions.md`, Section 3.2 precisely. Unlike the previous step, Daniel's account (`d.garcia`) is in active daily use, so the quieter **Shadow Credentials** technique is used instead of the (also-available) Reset Password option:

```text
certipy-ad shadow auto -target corp.apexfiend.lab -target-ip 172.16.20.100 \
    -dc-ip 172.16.20.100 -username m.rodriguez -password '<password>' \
    -account d.garcia
```

Certipy adds a certificate-backed key credential to `d.garcia`'s account, authenticates via PKINIT to obtain a TGT, recovers `d.garcia`'s NT hash via the key-trust mapping, and **restores the original Key Credentials afterward** — removing the added credential once the hash is recovered, leaving no persistent artifact on the account. This cleanup step mirrors the OPSEC discipline already noted elsewhere in this project (`05-databases/mssql03.md`, Section 4: re-disabling `xp_cmdshell` after testing).

Traffic for this step is routed from Kali through an obfuscated Ligolo tunnel via `DEV-LAPTOP` and `WK01`, reaching the domain controller directly without requiring every subsequent tool to run on a Windows host inside the domain.

## 8. Using d.garcia's Recovered Hash — Lateral Movement to WK02

From the RDP session on `WK01`, a local port-proxy (`netsh interface portproxy`) redirects traffic toward Kali via `DEV-LAPTOP`'s VPN tunnel address (`10.0.200.2`), allowing in-memory retrieval and execution of an AMSI bypass and Rubeus, entirely from WK01's own session:

```powershell
Invoke-Rubeus 'asktgt /user:d.garcia /domain:corp.apexfiend.lab /rc4:<NT-hash> /ptt'
```

The `/ptt` flag injects the resulting TGT directly into the current Windows logon session on `WK01` — `klist` confirms a cached TGT and service tickets for `d.garcia` against `DC02`/`DC03`. With that ticket active, `\\wk02\c$` is reachable, and **`PsExec64.exe` is used to obtain remote command execution on `WK02`**:

```text
hostname:  WK02
whoami:    corp\d.garcia
IP:        172.16.60.102
```

This completes the chain: code execution as `corp\d.garcia` on `WK02`, reached entirely through delegated Active Directory permissions and the organization's own weak onboarding password convention — no software exploit was required at any point in this document.

## 9. Chain Summary

```text
WK01 — corp\d.ramirez (from web-to-vpn.md)
      │  mail + Gitea recon → Mattermost access via ticket-email registration
      │  onboarding password pattern discovered and applied to Laura Sanchez
      ▼
l.sanchez (Laura) — GG-HelpDesk-L1
      │  Reset Password + account-state rights on OU=HelpDesk-L2
      │  (dormant account — low-risk action)
      ▼
m.rodriguez (Miguel) — GG-HelpDesk-L2, GG-WebOperations-Support
      │  msDS-KeyCredentialLink WriteProperty on OU=WebOperations
      │  (active account — Shadow Credentials chosen over Reset Password)
      ▼
Certipy shadow credentials → d.garcia's NT hash recovered (original state restored)
      │  Rubeus asktgt /ptt — ticket injected into the WK01 session
      ▼
WK02 — corp\d.garcia (PsExec)
```

## 10. Validation Status

| Item | Status |
|---|---|
| `d.ramirez`/`d.garcia` as the sole `GG-WebOperations` members | Tested and confirmed |
| Gitea authentication bypassing domain password via personal credential reuse | Tested and confirmed |
| `portal.apexfiend.lab` dual-resolution (WEB01 vs. WEB03) | Confirmed, not a documentation error |
| osTicket-to-Mattermost registration bypass | Tested and confirmed |
| Onboarding password-pattern weakness | Tested and confirmed (worked against Laura) |
| Laura's delegated rights over `OU=HelpDesk-L2`, re-confirmed live | Matches prior AD-level finding exactly |
| Miguel's account state and `GG-WebOperations-Support` membership | Tested and confirmed |
| Shadow Credentials chain, including credential cleanup | Tested and confirmed |
| Pass-the-ticket and lateral movement to WK02 as d.garcia | Tested and confirmed |

## 11. Related Documentation

- `web-to-vpn.md` — Precedes this document; ends with the `d.ramirez`/`WK01` foothold
- `sql-path.md` *(pending)* — Continues from `d.garcia`/`WK02` into `MSSQL01`
- `../02-active-directory/acl-design.md` — The ACL-level analysis this document executes against
- `../04-web-and-collaboration/gitea.md`, `web03.md` — Infrastructure referenced throughout