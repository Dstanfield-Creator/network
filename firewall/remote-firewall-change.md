# Remote Firewall Change Without Lockout

> Enable or change a host firewall (UFW or nftables) on a remote Linux host over SSH with a timed rollback armed first, so a mistake costs three minutes instead of a site visit.

**Status:** Active · **Updated:** 2026-10-08

## Scope

Applies to any Linux host you manage only over SSH: bare metal, a cloud VM, or a Proxmox guest. Examples use UFW; nftables equivalents are given where they differ.

## Why this needs a runbook

Enabling UFW from inside an existing SSH session can freeze that session even when the rule set is correct. The session's TCP connection already exists when the firewall loads, so conntrack picks it up mid-stream without the TCP window-scaling state it would have learned during the handshake. Once the window grows past what conntrack believes is valid, the kernel marks those packets INVALID, and UFW drops INVALID packets by default (the `ufw-before-input` chain carries a `-m conntrack --ctstate INVALID -j DROP` rule). The terminal hangs with no error, and the host may be unreachable.

A connection opened after the firewall is up is tracked from its SYN and does not have this problem. That is why the procedure tests with a brand new session and never trusts the one that made the change.

## Pre-checks

| Check | Command | Expected |
|---|---|---|
| Out-of-band path exists | Proxmox console, IPMI, cloud serial console, or `qm guest exec` | At least one works before you start |
| SSH port known | `ss -tlnp \| grep sshd` | 22 or your custom port is listening |
| Current ruleset saved | `sudo ufw status numbered`, `sudo nft list ruleset` | Saved copy to compare and roll back to |
| `systemd-run` available | `command -v systemd-run` | Path printed |
| No stale timer | `systemctl list-timers 'fw-deadman*'` | Nothing listed |

```bash
sudo ufw status verbose | sudo tee /root/ufw-before.txt
sudo nft list ruleset | sudo tee /root/nft-before.txt >/dev/null
ss -tlnp | grep -E 'sshd|:22 '
```

## Procedure

### 1. Arm the dead-man timer

Arm the rollback before touching anything. If you lock yourself out, the timer disables the firewall after 180 seconds and you get back in.

```bash
sudo systemd-run --unit=fw-deadman --on-active=180 /usr/sbin/ufw disable
systemctl list-timers fw-deadman.timer
```

For nftables, roll back to the saved ruleset instead:

```bash
sudo systemd-run --unit=fw-deadman --on-active=180 \
  /bin/sh -c 'nft flush ruleset && nft -f /root/nft-before.txt'
```

A helper script that wraps arm, test and disarm is at
https://github.com/Dstanfield-Creator/projects/tree/master/tools/firewall-deadman-switch

### 2. Stage the rules while the firewall is still off

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw limit 22/tcp comment 'ssh'
sudo ufw allow from 192.0.2.0/24 to any port 9100 proto tcp comment 'node_exporter'
sudo ufw show added
```

Adding rules changes nothing until `ufw enable` runs, so review `show added` carefully here.

### 3. Enable, out-of-band where possible

On a Proxmox guest with the QEMU guest agent running, make the change from the hypervisor so no SSH session is involved at all:

```bash
# On the Proxmox host
qm guest exec <vmid> -- ufw --force enable
qm guest exec <vmid> -- ufw status verbose
```

Other out-of-band paths: a cloud serial console, IPMI/iDRAC/iLO, or a console session you can afford to lose.

If no out-of-band path exists, run it in the SSH session anyway. The timer is already armed; the worst case is a 180-second wait.

```bash
sudo ufw --force enable
```

If this session freezes, leave it alone. Do not start disabling things by hand from a console. Go to step 4.

### 4. Test with a brand new SSH session

From your workstation, open a fresh connection. Do not reuse the existing one.

```bash
ssh -o ConnectTimeout=5 -o ControlMaster=no -o ControlPath=none \
    admin@host.example.com 'sudo ufw status verbose'
```

`ControlMaster=no` and `ControlPath=none` matter. If your SSH config multiplexes connections, a "new" ssh would otherwise ride the old TCP connection and prove nothing.

### 5. Disarm the timer only after the new session works

```bash
sudo systemctl stop fw-deadman.timer
systemctl list-timers fw-deadman.timer   # must be empty
```

If the new session fails, do nothing. Wait for the timer to fire, reconnect, and fix the rules. Working against the timer with partial information is how a second lockout happens.

## Verification

```bash
sudo ufw status verbose
sudo nft list ruleset | grep -c .
sudo journalctl -k --since '-10 min' | grep -i 'UFW BLOCK' | tail
ss -tlnp
```

From a second host, confirm that intended-closed ports are actually closed:

```bash
nc -zv -w 3 host.example.com 22      # open
nc -zv -w 3 host.example.com 5432    # refused or timeout
```

## Rollback

| Situation | Action |
|---|---|
| New session fails, timer still armed | Wait. The timer runs `ufw disable`; reconnect afterwards. |
| Timer already disarmed, a session still works | `sudo ufw disable`, then review the rules |
| No session at all | Proxmox console, or `qm guest exec <vmid> -- ufw disable` from the host |
| nftables | `sudo nft flush ruleset && sudo nft -f /root/nft-before.txt` |

Never leave the timer armed after a successful change. A forgotten timer disables the firewall three minutes later and nobody is told.

```bash
systemctl list-units 'fw-deadman*'
```

## nftables notes

- Keep the saved ruleset at `/root/nft-before.txt` and point the timer at it.
- Syntax-check with `sudo nft -c -f /etc/nftables.conf` before loading.
- The same INVALID-drop problem exists if a base chain has `ct state invalid drop`. The fresh-session test is still required.

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
