---
title: router-dhcp-dns-homelab
date: 2026-07-23
tags: ["linux", "homelab", "networking"]
---

This blog is part of an ongoing series of my homelab project [homelab](https://github.com/pranav10780/homelab)

# Router VM, DHCP, and DNS

## Goal

Stop relying on VirtualBox's invisible built-in NAT Network router, and
replace it with a VM (`router`) that you actually control: it hands out
addresses (DHCP), resolves names (DNS), and is the only path `server` and
`client` use to reach the internet. Host SSH access stays exactly as it
was, unaffected.

## Why this exists

Previously, every VM had one NIC on the VirtualBox `homelab` NAT Network.
DHCP, routing, and internet access for that network were all handled
silently by VirtualBox itself, this stage makes all things monitorable.

## Topology

Each of `server` and `client` now has **two** NICs:

- `enp0s3` — attached to the existing `homelab` NAT Network (10.0.2.0/24).
  Kept only for host SSH access via port forwarding. No longer used for
  internet access or DNS.
- `enp0s8` — attached to a new **Internal Network** called `lan`
  (10.0.3.0/24). This is the "real" network now — DHCP, DNS, and internet
  access all happen here, via `router`.

`router` also has two NICs:

- `enp0s3` — same `homelab` NAT Network, gives `router` its own internet
  access (this is what it forwards for everyone else) and SSH access from
  the host.
- `enp0s8` — static IP `10.0.3.1` on the `lan` network. Runs `dnsmasq`
  (DHCP + DNS) and does NAT/forwarding out to the internet via `enp0s3`.

## Part 1 — Routing (forwarding + NAT) on `router`

By default Linux does not pass traffic between two network interfaces,
it has to be explicitly enabled.

**Enable IP forwarding:**
```bash
sudo sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" | sudo tee -a /etc/sysctl.conf
sudo sysctl -p
```

**Enable NAT (MASQUERADE)** so that LAN-side traffic (10.0.3.x) is rewritten
to look like it came from the router's own WAN IP (10.0.2.x) when it goes out to the
internet, otherwise return traffic has nowhere valid to come back to:
```bash
sudo apt install iptables iptables-persistent -y
sudo iptables -t nat -A POSTROUTING -o enp0s3 -j MASQUERADE
sudo netfilter-persistent save
```

**Verify:**
```bash
cat /proc/sys/net/ipv4/ip_forward        # should print 1
sudo iptables -t nat -L POSTROUTING -n -v  # should show the MASQUERADE rule
```

## Part 2 — DHCP + DNS via dnsmasq on `router`

`dnsmasq` does both jobs.

```bash
sudo apt install dnsmasq -y
sudo cp /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
```

`/etc/dnsmasq.conf`:
```ini
interface=enp0s8
bind-interfaces

domain=homelab
expand-hosts

dhcp-range=10.0.3.50,10.0.3.150,12h

server=8.8.8.8
```

- `interface` / `bind-interfaces` — only serve DHCP/DNS on the LAN-facing
  NIC, never the WAN side.
- `dhcp-range` — pool of addresses handed out, with a 12 hour lease time.
- `domain` + `expand-hosts` — auto-registers whatever hostname a DHCP
  client announces, appending `.homelab` to it. This means `server` and
  `client` get working DNS names (`server.homelab`, `client.homelab`)
  automatically, with no manual `address=` entries required.
- `server=8.8.8.8` — anything not known locally (e.g. `google.com`) gets
  forwarded upstream.

```bash
sudo systemctl restart dnsmasq
sudo systemctl enable dnsmasq
```

**Where the records actually live:**
```bash
cat /var/lib/misc/dnsmasq.leases
```
This is the DHCP lease table - with `expand-hosts` on, dnsmasq also uses
it as the source of DNS answers for `*.homelab` names. There's no separate
zone file the way there would be with a heavier DNS server (bind9); this
is deliberate, dnsmasq stays lightweight by deriving DNS from DHCP state.

**Query it directly, bypassing any client's own resolver config:**
```bash
dig @10.0.3.1 server.homelab
dig @10.0.3.1 client.homelab
dig @10.0.3.1 router.homelab
```

## Part 3 — Making `server` and `client` use `router` for DHCP on boot

On both VMs, `/etc/network/interfaces` needs `enp0s8` brought up via DHCP
automatically at boot:

```
auto enp0s8
iface enp0s8 inet dhcp
```
```bash
sudo systemctl restart networking
```

## Part 4 — Fixing DNS resolver priority (the wrong server was being asked first)

**The problem:** each interface's DHCP lease can suggest a DNS server.
`enp0s3` (VirtualBox's `homelab` network) suggests its own built-in DNS
proxy at `10.0.2.3`; `enp0s8` suggests `10.0.3.1` (our router). Both ended
up listed in `/etc/resolv.conf`, with `10.0.2.3` first. Resolvers only try
the *next* nameserver on the list if the first one is unreachable or times
out — `10.0.2.3` was reachable and answered "no such name" for anything
under `.homelab` (since it has no idea that domain exists), so the second,
correct nameserver was never tried at all.

**The fix** — force `enp0s3` to contribute no DNS servers, on both `server`
and `client`, via `dhcpcd` (the DHCP client in use on this system):

```bash
sudo nano /etc/dhcpcd.conf
```
Add at the bottom:
```
interface enp0s3
static domain_name_servers=
```

Restart fully (a plain `--rebind` reuses stale lease data and won't apply
this correctly):
```bash
sudo dhcpcd -x
sudo rm /etc/resolv.conf
sudo dhcpcd
```

Verify:
```bash
cat /etc/resolv.conf   # should show only: nameserver 10.0.3.1
ping -c3 server.homelab
```

## Part 5 — Making internet traffic go through `router` only, not VirtualBox

Same shadowing problem as DNS, but for the **default route** — the "send
it here if I don't know where else" rule. Each interface's DHCP lease can
also offer a default route; both `enp0s3` (via VirtualBox, `10.0.2.1`) and
`enp0s8` (via `router`, `10.0.3.1`) were installing one, and depending on
route metrics, traffic could go out via either.

**The fix** — same `dhcpcd.conf` block as above, one more line:
```
interface enp0s3
static domain_name_servers=
nogateway
```

`nogateway` tells `dhcpcd` to ignore any default-route offer from this
interface's DHCP lease entirely.

```bash
sudo dhcpcd -x
sudo dhcpcd
ip route show
```

Expect exactly **one** `default via 10.0.3.1 dev enp0s8` line. The
directly-connected route to `10.0.2.0/24` (needed for host SSH) remains —
that's a separate, automatic route tied to the interface having an IP on
that subnet at all, unrelated to the default-route setting.

**Note:** on first attempt, `server`'s old default route via `10.0.2.1`
stuck around even after adding `nogateway`, because the existing DHCP
lease wasn't fully torn down on a soft rebind. Fixed with an explicit
release + re-request on that interface specifically:
```bash
sudo dhcpcd -k enp0s3
sudo dhcpcd enp0s3
```

## Verification (final state, both `server` and `client`)

```bash
ip route show
# expect exactly one default route, via 10.0.3.1 dev enp0s8

cat /etc/resolv.conf
# expect exactly one nameserver, 10.0.3.1

ping -c3 router.homelab
ping -c3 8.8.8.8
```
All should succeed, and `traceroute -I 8.8.8.8` should show `10.0.3.1` as
the first hop, confirming `router` — not VirtualBox — is the actual path
to the internet.

## See also

- [homelab](https://github.com/pranav10780/homelab) — this project
- [My projects](/projects) — tools I have built
- [Experience](/experience) — my background
