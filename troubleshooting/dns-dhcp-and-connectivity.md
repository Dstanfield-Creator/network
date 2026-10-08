# DNS, DHCP and Connectivity

> Layered diagnosis of "no network" and "can't reach X" on Windows and Linux, with the commands for each layer side by side and the failure signatures that point straight at the cause.

**Status:** Active · **Updated:** 2026-10-08

## Work the layers in order

Each layer depends on the one before it. Confirm a layer is good before moving up; skipping ahead is how DNS gets blamed for a dead cable.

```text
1. Link      - is the interface up with a carrier, or associated to Wi-Fi?
2. IP/DHCP   - does it have a valid address, mask, gateway and lease?
3. Gateway   - can it reach the default gateway?
4. DNS       - do names resolve, and to the right answers?
5. Path      - does traffic reach the destination network?
6. Service   - is the port open and the application answering?
```

Example values used below: client 192.0.2.57/24, gateway 192.0.2.1, DNS server 192.0.2.10, DHCP server 192.0.2.11, target `app.example.com` at 192.0.2.80, external resolver 198.51.100.53.

## Commands by layer

| Layer | Windows (cmd / PowerShell) | Linux |
|-------|----------------------------|-------|
| Link state | `Get-NetAdapter` | `ip -br link`; `nmcli device status` |
| Wi-Fi association | `netsh wlan show interfaces` | `iw dev wlan0 link`; `nmcli device wifi` |
| Address, mask, DHCP server, lease | `ipconfig /all` | `ip -br addr`; `nmcli -f IP4,DHCP4 device show eth0` |
| Release and renew a lease | `ipconfig /release` then `ipconfig /renew` | `sudo dhclient -v -r eth0 && sudo dhclient -v eth0`; `nmcli connection down "Wired" && nmcli connection up "Wired"` |
| Default route | `route print -4`; `Get-NetRoute -DestinationPrefix 0.0.0.0/0` | `ip route show default` |
| Reach the gateway | `ping 192.0.2.1` | `ping -c 4 192.0.2.1` |
| ARP / neighbour table | `arp -a`; `Get-NetNeighbor` | `ip neigh show` |
| Which resolver is in use | `ipconfig /all` (DNS Servers); `Get-DnsClientServerAddress` | `resolvectl status`; `cat /etc/resolv.conf` |
| Resolve a name | `nslookup app.example.com`; `Resolve-DnsName app.example.com` | `dig app.example.com`; `resolvectl query app.example.com` |
| Resolve against a specific server | `nslookup app.example.com 192.0.2.10`; `Resolve-DnsName app.example.com -Server 192.0.2.10` | `dig @192.0.2.10 app.example.com` |
| Flush the resolver cache | `ipconfig /flushdns`; `Clear-DnsClientCache` | `resolvectl flush-caches` |
| Trace the path | `tracert app.example.com`; `Test-NetConnection app.example.com -TraceRoute` | `traceroute app.example.com`; `mtr -rw app.example.com` |
| Test a TCP port | `Test-NetConnection 192.0.2.80 -Port 443` | `nc -zv 192.0.2.80 443`; `curl -v telnet://192.0.2.80:443` |
| Listening ports on a host | `netstat -ano \| findstr LISTEN`; `Get-NetTCPConnection -State Listen` | `ss -tlnp` |
| Host firewall state | `Get-NetFirewallProfile`; `netsh advfirewall show allprofiles` | `sudo nft list ruleset`; `sudo ufw status`; `sudo firewall-cmd --list-all` |
| Proxy settings | `netsh winhttp show proxy` | `env \| grep -i proxy` |

## A quick client-side sequence

Windows (PowerShell):

```powershell
Get-NetAdapter | Where-Object Status -eq Up
Get-NetIPConfiguration
Test-NetConnection 192.0.2.1
Resolve-DnsName app.example.com
Test-NetConnection app.example.com -Port 443
```

Linux:

```bash
ip -br link; ip -br addr; ip route show default
ping -c 3 192.0.2.1
resolvectl query app.example.com || dig app.example.com
nc -zv app.example.com 443
```

## Failure signatures

| What you see | What it almost always means | Next step |
|--------------|-----------------------------|-----------|
| Address 169.254.x.x (Windows APIPA) or no IPv4 address at all (Linux) | No DHCP answer. Link is up but no lease was offered. | Check the DHCP server, scope and relay (below); check the VLAN assigned to the switch port |
| Link shows down or no carrier | Cable, port, Wi-Fi association or a disabled adapter | Physical check; `Get-NetAdapter` or `ip -br link`; try another port |
| Valid address but the gateway does not ping | Wrong gateway, wrong VLAN, duplicate IP, or ICMP blocked on the gateway | Compare mask and gateway with a working host; `arp -a` to see whether the gateway MAC is learned |
| Gateway pings, `ping 192.0.2.80` works, `ping app.example.com` fails | DNS. Transport is fine, resolution is not. | `nslookup` or `dig` against the configured server and against a known-good one; check the resolver list |
| Name resolves to the wrong address | Stale cache, stale record, hosts-file override, or a split-brain zone | Flush the cache; check the hosts file; query the authoritative server with `dig @192.0.2.10 +norecurse` |
| `dig @192.0.2.10` times out, `dig @198.51.100.53` works | Internal resolver down or unreachable, or UDP/53 blocked on the path | Check the DNS service and the firewall rules on and in front of the resolver |
| Works on the LAN; over VPN works by IP but not by name | Split DNS. The VPN is not pushing the internal resolver or search domain, or the client prefers a public resolver. | Check the resolver and search suffix the VPN assigns; on Linux `resolvectl status` shows per-interface DNS |
| Over VPN some internal names work, others do not | Split-tunnel routes do not include that subnet, or the DNS suffix list is incomplete | `route print` or `ip route` for the destination prefix; add the route or suffix to the VPN profile |
| `tracert` stops at one hop, nothing after | Routing or firewall at that hop, or the next hop drops ICMP TTL-exceeded (not always a fault) | Test the actual TCP port; `mtr` to see the loss pattern |
| Port test fails but the server is up | Host firewall, service bound to localhost only, or a security group or ACL | `ss -tlnp` on the server; check the bind address; check firewall counters |
| Port opens, but the application errors | Not a network problem. Move to the service layer. | Application logs, TLS certificate, authentication |
| Intermittent loss or renewals every few minutes | Duplicate IP, rogue DHCP server, flapping link, lease exhaustion | `ip neigh` or `arp -a` for a changing MAC; look for a second DHCP server |

## Server-side DHCP checks

When clients show APIPA addresses, go to the DHCP server and the relay.

### Windows DHCP server

```powershell
Get-Service DHCPServer
Get-DhcpServerInDC                                 # is the server authorised in AD?
Get-DhcpServerv4Scope                              # state should be Active
Get-DhcpServerv4ScopeStatistics                    # watch Free and PercentageInUse
Get-DhcpServerv4Lease -ScopeId 192.0.2.0 | Measure-Object
Get-DhcpServerv4Failover                           # if failover is configured
Get-WinEvent -LogName "Microsoft-Windows-Dhcp-Server/Operational" -MaxEvents 50
```

Lease exhaustion shows as `Free` near zero in the scope statistics and event ID 1020 in the System log. Fixes: shorten the lease duration, widen the scope, or remove stale reservations. A scope that is inactive, or a server that is not authorised in AD, will not answer at all.

### Linux DHCP server (ISC dhcpd or Kea)

```bash
systemctl status isc-dhcp-server        # or kea-dhcp4-server
journalctl -u isc-dhcp-server -n 100
grep -c "^lease " /var/lib/dhcp/dhcpd.leases
dhcpd -t -cf /etc/dhcp/dhcpd.conf       # syntax check
```

### DHCP relay

Clients on a different subnet from the DHCP server rely on the router or L3 switch forwarding their broadcasts as unicast. On most Cisco-style devices the relay is configured per interface:

```text
interface Vlan20
 ip address 192.0.2.1 255.255.255.0
 ip helper-address 192.0.2.11
```

Checks:

- The helper address is present on the client's VLAN interface and points at the current DHCP server, not a decommissioned one.
- The DHCP server has a scope matching the relay interface's subnet (192.0.2.0/24 here); the server picks the scope from the `giaddr` field, not from its own address.
- The firewall between relay and server allows UDP/67 from the relay address.
- A capture on the server (`tcpdump -ni eth0 port 67 or port 68`) shows Discover packets arriving; if it does not, the problem is before the server.

## Related

- [Troubleshooting methodology](https://github.com/Dstanfield-Creator/general-it/blob/master/docs/troubleshooting-methodology.md)
- [Windows and Linux command equivalents](https://github.com/Dstanfield-Creator/general-it/blob/master/reference/windows-linux-command-equivalents.md)

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
