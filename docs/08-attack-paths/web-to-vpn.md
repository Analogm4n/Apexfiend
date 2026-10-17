# Apexfiend — WEB01 to VPN (DEV-LAPTOP)

## 1. Purpose

This document continues directly from `initial-access.md`, Section 7: a live forwarded SSH agent socket on `WEB01` has just been used to reach `DEV-LAPTOP` (`10.0.100.12`) as `developer`. It narrates the credential-harvesting and pivoting chain that follows, ending with authenticated access to `WK01` inside the corporate domain.

## 2. DEV-LAPTOP Enumeration

```text
whoami:    dev-laptop\developer
ipconfig:  10.0.100.12 (external/development network — see 01-architecture/network-architecture.md)
```

Enumeration of the `developer` user's profile surfaces several items of interest:

- **`r.andrews-vpn.ovpn`** on the Desktop — a VPN client profile belonging to a *different* user, `r.andrews`. This is the same `r.andrews` already present as a local pfSense GUI user (`03-network-services/pfsense.md`, Section 3) — confirming that a pfSense-level account also has lab VPN access, distinct from the domain-integrated OpenVPN remote-access configuration already documented.
- **Mozilla Firefox**, installed and used, with a saved-password store.
- **`Credentials.kdbx`**, a KeePass database, in Documents.
- **PSTools** (Sysinternals), downloaded but not yet used at this stage.

## 3. Credential Harvesting — Firefox and KeePass

The Firefox profile directory is archived and exfiltrated to the attacker's own host via `scp` **initiated outbound from DEV-LAPTOP** — deliberately reversing the usual direction, since an outbound connection from an internal/DMZ-adjacent host is far less likely to be blocked by firewall egress policy than an inbound one. `firefox_decrypt` run offline against the extracted profile recovers multiple saved site credentials, including `d.ramirez@apexfiend.lab`'s mail credentials and several personal-site credentials for the same person — `d.ramirez` is the actual human behind the `developer`/`dev` activity on this machine.

**Credential reuse across the harvested secrets opens the KeePass database**: one of the Firefox-recovered passwords successfully unlocks `Credentials.kdbx`, revealing domain credentials for `d.ramirez`, a reference to workstation `WK01`, and further Outlook credentials. This is a realistic password-reuse finding — the same weakness repeatedly exploited across the lab's design (`02-active-directory/acl-design.md`'s Help Desk chain relies on a related principle: reachable, reusable access once a single credential set is obtained).

## 4. VPN Access as r.andrews

The `r.andrews-vpn.ovpn` profile found in Section 2 is adapted for non-interactive, GUI-less use on the Windows host (no desktop session is available over the current shell) by appending an `auth-user-pass` directive pointing at a credentials file, populated with `r.andrews`'s own recovered credentials. The OpenVPN Connect client is installed as a service and started with this profile, successfully establishing a tunnel:

```text
VPN adapter IPv4 address: 10.0.200.2
```

This falls within the documented VPN tunnel network, `10.0.200.0/24` (`03-network-services/vpn-config.md`), confirming the connection is the lab's genuine remote-access VPN — not a separate or decoy network.

## 5. Reaching WK01

With the VPN tunnel active, a reverse SSH port-forward is established from DEV-LAPTOP back to the attacker's host, forwarding local port 3389 to `172.16.60.104:3389` — `WK01`'s RDP port, reachable only now that DEV-LAPTOP has VPN-routed access into `172.16.60.0/24` (the sole network the VPN pushes, per `vpn-config.md`, Section 4). RDP is then reached from the attacker's own machine through that tunnel, authenticating with `d.ramirez`'s domain credentials recovered from KeePass in Section 3:

```text
whoami:    corp\d.ramirez
hostname:  WK01
IP:        172.16.60.104
```

This completes initial access into the corporate domain: an authenticated interactive session as a real domain user, on a real domain-joined workstation, reached without ever possessing a plaintext credential that was not itself recovered from the compromised chain.

## 6. Chain Summary

```text
WEB01 (www-data, container)
      │  filename argument-injection container escape
      ▼
WEB01 (dev, host)
      │  dangling git commit → Gitea token → private repo → DEV-LAPTOP identified
      │  live forwarded SSH agent socket
      ▼
DEV-LAPTOP (developer)
      │  Firefox credential harvest → KeePass unlock (password reuse)
      │  r.andrews' VPN profile + recovered credentials
      ▼
VPN tunnel (10.0.200.0/24) → routes only to VLAN 60
      │  reverse SSH port-forward (RDP)
      │  d.ramirez's domain credentials (from KeePass)
      ▼
WK01 — corp\d.ramirez
```

## 7. Validation Status

| Item | Status |
|---|---|
| DEV-LAPTOP enumeration findings | Tested and confirmed |
| Firefox credential extraction | Tested and confirmed |
| KeePass unlock via Firefox-recovered password reuse | Tested and confirmed |
| VPN connection as `r.andrews`, tunnel network confirmed | Tested and confirmed |
| Reverse SSH tunnel + RDP to WK01 as `d.ramirez` | Tested and confirmed |

## 8. Related Documentation

- `initial-access.md` — Precedes this document; WEB01 compromise through the forwarded-agent pivot
- `helpdesk-escalation.md` *(pending)* — Continues from WK01: Laura → Miguel → Daniel
- `../03-network-services/vpn-config.md`, `pfsense.md` — VPN and `r.andrews` account detail
- `../00-overview/attack-path.md` — Stage 1 (kept intentionally general; this document is the detailed counterpart)