# Apexfiend — Certificate Templates

## 1. Purpose

This document records the verified configuration of the lab's certificate templates, with particular focus on `UserTemporary` — the template central to the attack path's PKI stage — compared against the default `User` template it was copied from.

All flag values below were decoded precisely from their raw LDAP attribute values (not estimated), since this is foundational to correctly identifying which certificate-abuse technique applies.

## 2. UserTemporary — Attributes

```powershell
Get-ADObject -LDAPFilter "(cn=UserTemporary)" -SearchBase $Base -Properties * 
```

```text
DistinguishedName:   CN=UserTemporary,CN=Certificate Templates,CN=Public Key Services,CN=Services,CN=Configuration,DC=apexfiend,DC=lab
Created:              2026-10-13 (recently created relative to the rest of the lab)
msPKI-Certificate-Name-Flag:   -1509949440  (unsigned: 0xA6000000)
msPKI-Enrollment-Flag:          41  (0b101001)
msPKI-Private-Key-Flag:          16842768  (0x01010010)
msPKI-Minimal-Key-Size:          2048
pKIExtendedKeyUsage:              Client Authentication (1.3.6.1.5.5.7.3.2),
                                   Secure Email (1.3.6.1.5.5.7.3.4),
                                   Encrypting File System (1.3.6.1.4.1.311.10.3.4)
```

### Decoded: msPKI-Certificate-Name-Flag (0xA6000000)

| Bit | Flag | Set? |
|---|---|---|
| `0x00000001` | `ENROLLEE_SUPPLIES_SUBJECT` | **No** |
| `0x00010000` | `ENROLLEE_SUPPLIES_SUBJECT_ALT_NAME` | No |
| `0x02000000` | `SUBJECT_ALT_REQUIRE_UPN` | Yes |
| `0x04000000` | `SUBJECT_ALT_REQUIRE_EMAIL` | Yes |
| `0x20000000` | `SUBJECT_REQUIRE_EMAIL` | Yes |
| `0x80000000` | `SUBJECT_REQUIRE_DIRECTORY_PATH` | Yes |

**`ENROLLEE_SUPPLIES_SUBJECT` is not set.** The requester cannot specify an arbitrary subject — the certificate's identity is built from the enrollee's own AD attributes, exactly as the default `User` template behaves. **This template is not vulnerable to the classic ESC1 technique** (arbitrary subject + client authentication EKU + low-privilege enrollment), contrary to how the equivalent template was described in `pki-documentation/templates/legacy-and-test-templates.md` (as `LEGACY-WebEnrollment-Compat`, an ESC1-style misconfiguration). That document needs correcting once the real technique is confirmed (see Section 4).

### Decoded: msPKI-Enrollment-Flag (41 = 0b101001)

| Bit | Flag | Set? |
|---|---|---|
| `0x00000001` | `PUBLISH_TO_DS` | Yes |
| `0x00000002` | `PEND_ALL_REQUESTS` (manager approval required) | **No** |
| `0x00000008` | `INCLUDE_SYMMETRIC_ALGORITHMS` | Yes |
| `0x00000020` | `AUTO_ENROLLMENT` | Yes |
| `0x00080000` | `NO_SECURITY_EXTENSION` (ESC9-relevant) | No |

**No manager approval is required to enroll** — a request against this template, by anyone with the Enroll right, is issued automatically.

### Decoded: msPKI-Private-Key-Flag (16842768 = 0x01010010)

`EXPORTABLE_KEY` is set — the issued certificate's private key can be exported after issuance. `REQUIRE_PRIVATE_KEY_ARCHIVAL` and `STRONG_KEY_PROTECTION_REQUIRED` are not set.

## 3. UserTemporary — Access Control

```powershell
(Get-Acl "AD:\CN=UserTemporary,...").Access
```

| Identity | Rights | Notes |
|---|---|---|
| `NT AUTHORITY\Authenticated Users` | GenericRead | Standard — read-only for everyone |
| `APEXFIEND\Domain Admins`, `APEXFIEND\Enterprise Admins`, `CORP\admin` | Full control | Template management |
| `APEXFIEND\Domain Admins`, `APEXFIEND\Domain Users`, `APEXFIEND\Enterprise Admins` | Enroll (ExtendedRight) | **Inherited from the Certificate Templates container — identical on the default `User` template too (Section 5); not specific to `UserTemporary`** |
| `CORP\PKI-Operations` | ReadProperty, GenericExecute | **Read-only — no Enroll right.** Matches `j.derek`'s documented role (`02-active-directory/delegated-permissions.md`, deferred item now resolved) |
| `CORP\m.ortega` | ReadProperty, GenericExecute, **Enroll (ExtendedRight)** | **The one individually-named identity with enrollment rights on this template** |

**The meaningful difference between `UserTemporary` and the default `User` template is two added ACEs: `PKI-Operations` (read-only) and `m.ortega` (read + Enroll).** Everything else — including the broad `APEXFIEND\Domain Users` Enroll grant — is inherited and identical between the two templates, confirming it is a default forest-level condition, not something specific to `UserTemporary`'s design.

**This resolves the `j.derek` open item from `02-active-directory/delegated-permissions.md`:** `PKI-Operations` (and therefore `j.derek`) has confirmed read-only access to this template — no enrollment or modification rights, exactly as the lab's design intended.

## 4. The Real Technique: ESC8, Not ESC1

Combining Sections 2 and 3:

- `m.ortega` has an individually-granted **Enroll** right on a template with **Client Authentication** EKU and **no manager approval required**.
- The template does **not** allow the requester to supply an arbitrary subject (`ENROLLEE_SUPPLIES_SUBJECT` is unset) — so an attacker cannot simply request a certificate *as* `m.ortega` by naming him in the request.
- What an attacker *can* do is relay `m.ortega`'s own NTLM authentication to the CA's HTTP enrollment endpoint, causing the CA to issue a certificate **on `m.ortega`'s behalf**, using his own relayed authentication rather than a spoofed subject.

This is the **ESC8** technique (NTLM relay to AD CS HTTP Web Enrollment), and it matches `00-overview/attack-path.md` Stage 7's description of an NTLM relay against the certificate enrollment HTTP endpoint precisely. **Pending confirmation from the operator** on two points before `pki-documentation/` is corrected: (1) that ESC8 is indeed the intended technique, and (2) the role of the separately-published template literally named `esc8` (Section 6) — whether it is the actual template exploited, a decoy, or something else.

## 5. User — Attributes and ACL (Comparison Baseline)

```text
DistinguishedName: CN=User,CN=Certificate Templates,...
```

ACL: `Authenticated Users` (GenericRead), `Domain Admins`/`Enterprise Admins` (Full control, template management), `Domain Admins`/`Domain Users`/`Enterprise Admins` (Enroll). No entries for `PKI-Operations` or `m.ortega` — confirming those two ACEs are specific additions on `UserTemporary`, not inherited defaults shared by every template.

## 6. Open Item — the `esc8` Template

A separate template literally named `esc8` is also published on the CA (`ca02.md`, Section 5). Its configuration has not yet been queried. Whether this is the template actually exploited in the designed attack path (in which case `UserTemporary`'s role needs reconsidering), a deliberate decoy alongside `test`/`test-p`, or unrelated noise, is an open question for the operator to resolve before this document and `pki-documentation/` are finalized.

## 7. Validation Status

| Item | Status |
|---|---|
| `UserTemporary` flag values, precisely decoded | Verified |
| `UserTemporary` ACL | Verified |
| `User` (baseline) ACL | Verified |
| `PKI-Operations`/`j.derek` read-only access | Verified — resolves prior open item |
| `m.ortega`'s individual Enroll grant | Verified |
| ESC8 as the applicable technique (vs. ESC1 as currently documented in `pki-documentation/`) | Strongly indicated by data; pending operator confirmation |
| The separate `esc8`-named template's role | Not yet queried |

## 8. Related Documentation

- `ca02.md` — CA-level configuration and full template list
- `02-active-directory/delegated-permissions.md` — The `j.derek`/`PKI-Operations` item this document resolves
- `../00-overview/attack-path.md` — Stage 6 (PKI discovery) and Stage 7 (certificate abuse / NTLM relay)
- `../../pki-documentation/` *(separate repository)* — Needs correction pending the ESC1 → ESC8 confirmation