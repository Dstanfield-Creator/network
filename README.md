# Network

> Zero-trust remote access, campus design, firewalls, and the connectivity troubleshooting that goes with them. This is the Network department of my GitHub: every network-related build, tool, runbook and reference lives here.

**Status:** Active · **Updated:** 2026-10-08

## Structure

```
├── remote-access/
│   ├── tailscale-remote-access/          # WireGuard mesh, MagicDNS names as SSH handles, 1Password agent, no port-forwards
│   └── pi-travel-router/                 # OpenWrt pocket router with a Tailscale exit node back to the lab
├── design/
│   └── cisco-enterprise-network-design/  # Multi-VLAN campus: core SVIs, DHCP relay, hardened access ports, ASA edge
├── firewall/
│   ├── firewall-deadman-switch/          # fw-deadman: systemd timer that rolls back a remote firewall change
│   ├── remote-firewall-change.md         # Runbook: arm the rollback, change out-of-band, test from a fresh session
│   └── ufw-baseline-linux.md             # Default-deny UFW baseline for headless Debian/Ubuntu servers
├── troubleshooting/
│   └── dns-dhcp-and-connectivity.md      # Layered "no network" diagnosis on Windows and Linux
├── case-studies/
│   └── network-security-enhancement/     # Professional: segmentation, Fortinet firewall/IDS, backup redesign
├── CONTRIBUTING.md                       # Sanitisation rules
└── LICENSE
```

## Contents

### Remote access

| Item | Summary | Status |
|---|---|---|
| [Tailscale Remote Access](./remote-access/tailscale-remote-access/) | No port-forwards: WireGuard mesh, tailnet names as stable SSH handles, keys served by the 1Password SSH agent, access matrix and failure modes | Active |
| [Raspberry Pi Travel Router](./remote-access/pi-travel-router/) | OpenWrt on a Pi with a Tailscale exit node, so every device on the road connects once to a network you control | Build guide |

### Design

| Item | Summary | Status |
|---|---|---|
| [Cisco Enterprise Network Design](./design/cisco-enterprise-network-design/) | VLAN plan, core SVIs and DHCP relay, hardened access ports, ASA NAT and policy, verification commands | Completed |

### Firewall

| Item | Summary | Status |
|---|---|---|
| [Firewall Dead-Man Switch](./firewall/firewall-deadman-switch/) | `fw-deadman`: a systemd timer that disables the firewall if the change locks you out, plus the procedure around it | Active |
| [Remote Firewall Change Without Lockout](./firewall/remote-firewall-change.md) | Runbook for enabling or changing UFW/nftables over SSH: arm the rollback, change out-of-band, test from a new session, disarm | Active |
| [UFW Baseline for Headless Servers](./firewall/ufw-baseline-linux.md) | Default deny, management-subnet SSH, Tailscale interface allow, SSH rate limiting, rollback timer for remote enables | Active |

### Troubleshooting

| Item | Summary |
|---|---|
| [DNS, DHCP and Connectivity](./troubleshooting/dns-dhcp-and-connectivity.md) | Layered diagnosis of "no network" on Windows and Linux, failure signatures, server-side DHCP relay checks |

### Case studies

| Item | Summary | Status |
|---|---|---|
| [Network Optimisation & Security Enhancement](./case-studies/network-security-enhancement/) | Assessment, Fortinet firewall/IDS, Veeam/Acronis backup, audits and PowerShell automation across an MSP estate (2022–2025) | Completed |

## Related

- Attack-side network reference in [cyber-resources](https://github.com/Dstanfield-Creator/cyber-resources): [Firewall Configuration](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/docs/firewall-configuration.md) · [Network Segmentation](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/docs/network-segmentation.md) · [Network Protocol Attacks](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/docs/network-protocol-attacks.md) · [Network Scanning](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/techniques/network-scanning.md) · [Port Scanning](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/techniques/port-scanning.md) · [DNS Enumeration](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/techniques/dns-enumeration.md) · [Protocol Analysis](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/techniques/protocol-analysis.md) · [Wireshark](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/tools/wireshark.md) · [Nmap](https://github.com/Dstanfield-Creator/cyber-resources/blob/master/tools/nmap.md)
- The hosts these rules protect: [lab-ops](https://github.com/Dstanfield-Creator/lab-ops), where the Ansible `common` role applies the UFW baseline
- Write-up: [The UFW rule was correct and still locked me out](https://dstanfield-creator.github.io/writeups/ufw-lockout.html), the conntrack lesson behind the dead-man switch

## Conventions

- Real IPs, MACs, usernames, tailnet names and secrets are replaced with documentation placeholders (RFC 5737 `192.0.2.0/24`, `example.ts.net`). VM IDs and hostnames are real.
- Scripts are the versions actually in use, with credentials removed.
- Professional case studies describe how the work was structured and what it delivered; client details are omitted.

See [CONTRIBUTING.md](./CONTRIBUTING.md) for the full sanitisation rules.

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
