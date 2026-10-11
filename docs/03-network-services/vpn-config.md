# Apexfiend — VPN Configuration

## 1. Purpose

This document records the verified OpenVPN server configuration on pfSense, excluding all key material (CA private key, server certificate/key, client certificates/keys, and the TLS-auth static key), which must never be committed to the public repository.

## 2. Server Configuration

```text
Mode:              Remote Access (SSL/TLS + User Auth)
Interface:         WAN
Protocol / Port:   UDP4 / 1194
Device mode:       tun
Authentication:    Local Database
```

## 3. Cryptographic Configuration

```text
Data ciphers (negotiated): AES-256-GCM, AES-128-GCM, CHACHA20-POLY1305
Data cipher fallback:       AES-256-CBC
Digest:                      SHA256
Diffie-Hellman parameters:   2048-bit
TLS-auth:                    Enabled (static key — not reproduced here)
Compression:                  Disabled
```

## 4. Network Configuration

```text
Tunnel network:   10.0.200.0/24
Local network (pushed to clients): 172.16.60.0/24
Topology:          subnet
DNS domain pushed: corp.apexfiend.lab
DNS server pushed: 127.0.0.1  (pfSense itself, which forwards per dns.md Section 3)
Exit notify:        enabled
Inactive timeout:   300 seconds
```

**Only `172.16.60.0/24` (the workstation VLAN) is pushed to connecting clients as a local route.** Combined with the permissive `pass any` firewall rule on the `openvpn` interface documented in `pfsense.md`, Section 5, this confirms that VPN client reach is enforced by the **pushed route**, not by firewall policy — a client is only ever told how to reach VLAN 60, and the firewall does not independently restrict it to that segment. This satisfies the intent described in `segmentation.md`, Section 4, though via a different mechanism than originally assumed (route-based rather than firewall-based restriction).

## 5. Certificates and Keys

The live configuration includes:

- An internal Certificate Authority (`OpenVPN_CA`)
- A dedicated OpenVPN server certificate (distinct from pfSense's default GUI certificate)
- At least one issued client certificate

All private key material associated with these was present in the original configuration export and has been deliberately excluded from this document and from the repository. If any of this material was ever committed to a public location, it should be treated as compromised and rotated — the CA reissued, the server certificate reissued, and any client certificates revoked and reissued.

## 6. Validation Status

| Item | Status |
|---|---|
| Server mode, protocol, port | Verified |
| Cryptographic configuration | Verified |
| Tunnel network and pushed local route | Verified |
| Firewall enforcement model (route-based, not rule-based) | Verified — cross-referenced with `pfsense.md` |
| Number and identity of issued client certificates | Not fully inventoried in this pass |

## 7. Related Documentation

- `pfsense.md` — Section 5 covers the firewall-level rule for the `openvpn` interface
- `segmentation.md` — Section 4 describes the intended VPN access boundary this configuration implements
- `01-architecture/network-architecture.md` — VPN's role in the overall access model