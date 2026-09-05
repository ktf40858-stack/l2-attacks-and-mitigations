# Attack 4 — CAM table overflow (MAC flooding)

**ATT&CK:** T1040 Network Sniffing
**Mitigation:** [Port security](../mitigations/03-port-security.md)

## What it does

A switch learns which MAC lives on which port and stores the bindings in its CAM table, a
fixed-size hardware table. Flood the switch with frames from thousands of forged source MACs
and the table fills. Once full, the switch **fails open**: a frame for a destination it can no
longer fit in the table is flooded out every port, like a hub. The attacker on any port then
sees traffic destined for every other port — passive sniffing of the whole VLAN.

Fail-open is the dangerous design choice here. A switch that fails closed would drop; a switch
that fails open leaks.

## How to run it

```bash
macof -i eth0            # generates ~155,000 forged frames per second
```

`macof` fills a typical access-switch CAM table in single-digit seconds.

## What you see when it works

On the switch, the MAC table saturates:

```
S1# show mac address-table count
  Dynamic Address Count:  8192      # table is at capacity, all forged
  Total MAC Addresses:    8192
```

On the attacker, traffic that should never reach this port starts arriving:

```bash
tcpdump -i eth0 not arp        # now shows unicast between OTHER hosts on the VLAN
```

## The fingerprint it leaves

- **Thousands of source MACs learned on a single access port** in seconds. A wall jack sees
  one host; ten thousand MACs on it is not a host, it is `macof`.
- **CAM table at capacity** with mostly random-looking MACs.
- **Port-security violations**, once the cap is in place — the counter jumps the instant the
  flood starts.

## Why it works, in one line

An access port with no MAC limit will learn every source it sees. Port security caps the count
and shuts the port when the cap is exceeded, so the flood shuts itself out on the first extra MAC.
