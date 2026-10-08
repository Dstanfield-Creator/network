# Raspberry Pi Travel Router

> A pocket router that turns hotel Wi-Fi into a private, firewalled network for all your devices, with an encrypted tunnel back to the home lab for a trusted exit.

**Category:** Homelab · **Status:** Build guide · **Hardware:** Raspberry Pi 4/5 · **Software:** OpenWrt + Tailscale

## Why

Hotel, airport and conference networks are hostile by default: captive portals, client isolation that breaks Chromecasts and consoles, and no idea who else is on the segment. A travel router means every device connects once, to a network you control, and the router handles the hostile side. Adding a Tailscale exit node at home means traffic leaves from the home connection, not the hotel's.

## Bill of materials

| Part | Notes |
|---|---|
| Raspberry Pi 4 (2 GB+) or Pi 5 | Pi 5 needs the active cooler in a closed case |
| USB Wi-Fi adapter with AP support | e.g. an `mt76`/`rtl8812au`-based dual-band adapter; the onboard radio becomes the AP, the USB adapter the WAN client (or vice-versa) |
| microSD (A2, 32 GB) or USB SSD | OpenWrt is tiny; the A2 rating matters for log writes |
| USB-C PD power bank, 45 W+ | Pi 5 is picky about 5 V / 5 A negotiation |
| Short Cat6 patch lead | For hotels with Ethernet, which is the best WAN option |
| Case with room for the adapter | Flirc or Argon-style passive cases work for the Pi 4 |

## Network design

```mermaid
flowchart LR
  HOTEL["Hotel Wi-Fi / Ethernet"] --> WAN["WAN - wlan1 or eth0"]
  CLIENTS["Phone / laptop / console"] --> AP["LAN AP - wlan0"]
  WAN --> OW["OpenWrt<br/>firewall zones, NAT,<br/>DHCP, DNS filtering"]
  AP --> OW
  OW --> TS["tailscale0<br/>exit node at home (optional)"]
```

- **Two radios:** one associates to the hotel network as a client (`wwan`), the other broadcasts your own SSID. Ethernet WAN is preferred when available.
- **Firewall:** OpenWrt's default `wan → lan` reject policy; LAN devices see each other (fixes client-isolation problems) and nothing on the hotel side sees them.
- **DNS:** `dnsmasq` with an `adblock` list, upstream over DoT to a resolver you trust, or through the tunnel to the lab's resolver.
- **Exit node:** `tailscale up --exit-node=<home-node> --exit-node-allow-lan-access` sends everything through the lab's connection. Latency cost from a hotel is usually 20–40 ms; worth it on anything untrusted.

## Build

```bash
# 1. Flash OpenWrt (bcm27xx/bcm2711 for Pi 4, bcm2712 for Pi 5) with the Raspberry Pi Imager
# 2. First boot over Ethernet to 192.168.1.1, set root password, then:

opkg update
opkg install kmod-usb-net-rtl8152 luci wpad-wolfssl travelmate luci-app-travelmate \
             tailscale luci-app-adblock adblock

# 3. Radios: wlan0 = your AP, wlan1 = USB adapter as WAN client
uci set wireless.radio0.disabled='0'
uci set wireless.default_radio0.ssid='travelnet'
uci set wireless.default_radio0.encryption='sae-mixed'
uci set wireless.default_radio0.key='<long passphrase>'
uci commit wireless

# 4. travelmate manages the client side: it scans, handles captive portals,
#    and keeps a list of known hotel networks in order of preference
uci set travelmate.global.trm_enabled='1'
uci commit travelmate; /etc/init.d/travelmate restart

# 5. Firewall: wwan joins the wan zone; lan stays default (accept in, reject wan→lan)
uci add_list firewall.@zone[1].network='wwan'
uci commit firewall; /etc/init.d/firewall restart

# 6. Tailscale
/etc/init.d/tailscale enable && /etc/init.d/tailscale start
tailscale up --accept-dns=false --exit-node=<home-node> --exit-node-allow-lan-access
# add tailscale0 to the wan zone so LAN clients are NATed through it
```

### Captive portals

Travelmate's captive-portal detection opens the portal page in a built-in handler; failing that, connect a laptop to the travel SSID, browse to any `http://` site, and complete the portal. The router's MAC is what gets authorised, so every device behind it is covered at once.

## Checks before every trip

- [ ] `opkg list-upgradable` and update, at home, where there is time to fix it
- [ ] Known networks list pruned; AP passphrase rotated if it was shared last trip
- [ ] Exit node reachable: `tailscale status` shows the home node online
- [ ] `adblock` lists updated
- [ ] Power bank charged; the Pi 5 will brown-out silently on a weak supply

## Lessons learned

- **Ethernet WAN when offered, always.** Hotel Wi-Fi 5 GHz client + 2.4 GHz AP works, but Ethernet removes the flakiest link.
- **Don't let the hotel DHCP hand you a range that collides with the LAN.** Using `10.42.0.0/24` for the LAN avoids every hotel `192.168.x.x`.
- **The exit node is optional per device.** Streaming boxes can bypass it; laptops and phones go through it.

## Skills demonstrated

OpenWrt · Wi-Fi client/AP design · firewall zoning and NAT · DNS filtering · Tailscale exit nodes · travel OPSEC

## Related

- [Tailscale Remote Access](../tailscale-remote-access/) — the home side of the tunnel

---

**Author:** Danny Stanfield · Perth, WA  
**License:** MIT
