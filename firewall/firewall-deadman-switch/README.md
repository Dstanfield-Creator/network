# Firewall Dead-Man Switch

> Never change a remote host's firewall without a timer that undoes it if you lock yourself out.

**Category:** Tools · **Status:** Active (standard procedure for every remote firewall change) · **Language:** Bash + systemd

## The incident that produced it

Enabling UFW for the first time on a headless VM, over SSH, with a correct rule set (`allow 22/tcp from <LAN>`). The moment `ufw enable` ran, the SSH session froze. The rules were right. The problem was **connection tracking**: the new firewall picked up the existing SSH connection mid-stream without having seen its handshake, so it did not know the TCP window scaling in use, classified the packets as `INVALID`, and UFW's default `INVALID` drop ate the session. A fresh connection would have worked, but there was no way to open one from inside a frozen one, and the VM had no console session open.

What saved it was a transient systemd timer armed thirty seconds earlier:

```bash
sudo systemd-run --unit=ufw-deadman --on-active=180 /usr/sbin/ufw disable
```

Three minutes later UFW turned itself off and SSH came back. The procedure below is what came out of it.

## The procedure

1. **Arm** the rollback on the target host (default: disable UFW in 180 s):
   ```bash
   sudo fw-deadman arm            # or: sudo fw-deadman arm 300 -- nft flush ruleset
   ```
2. **Make the change**, preferably **out-of-band**, not inside the SSH session you depend on. On a Proxmox guest with the QEMU agent:
   ```bash
   qm guest exec <vmid> -- ufw --force enable
   ```
   On bare metal, use a console, IPMI, or a second session you are prepared to lose.
3. **Open a brand-new SSH session** to the host. The existing one proves nothing; the new one is the test.
4. **Disarm** only from that new session:
   ```bash
   sudo fw-deadman disarm
   ```
5. If the new session fails: do nothing, wait for the timer, and let it roll back. Do not retry on top of a half-applied change.

## `fw-deadman`

A thin wrapper over `systemd-run` so the procedure is one word and the unit name is always the same:

```
sudo fw-deadman arm [SECONDS] [-- ROLLBACK_CMD ...]   # default 180s, default rollback: ufw disable
sudo fw-deadman disarm                                # cancel the timer
     fw-deadman status                                # show the pending timer, if any
```

It refuses to double-arm, and `disarm` tells you if the rollback has already fired so you know to check the firewall state rather than assume it is up.

Install: `sudo install -m 755 fw-deadman /usr/local/sbin/`

## Why systemd-run and not `sleep … &`

- Survives the SSH session dying, which is the whole point. A backgrounded shell job is killed with its session unless you remember `nohup`/`setsid` every time.
- Visible: `systemctl list-timers` shows exactly when it fires.
- Cancellable from any session with one command.
- Logged in the journal when it fires, so there is a record of the rollback.

## Related

- [Paperclip AI Agents](https://github.com/Dstanfield-Creator/ai-automation/tree/main/paperclip-ai-agents/) — the VM where this was learned the hard way
- [Tailscale Remote Access](../../remote-access/tailscale-remote-access/) — the access path this protects

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
