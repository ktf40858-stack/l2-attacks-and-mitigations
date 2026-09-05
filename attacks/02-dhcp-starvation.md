# Attack 2 — DHCP starvation

**ATT&CK:** T1498 Network Denial of Service
**Mitigation:** [Port security + DHCP snooping](../mitigations/02-dhcp-snooping.md)

## What it does

A DHCP server hands out addresses from a finite pool. If an attacker requests addresses faster
than the pool can be replenished — each request forged from a different source MAC — the pool
empties. Legitimate clients that boot afterward get no address and cannot reach the network.

Starvation is often step one of a two-step attack: empty the real DHCP server, then stand up a
[rogue one](03-rogue-dhcp.md) that answers instead, handing clients the attacker's IP as their
default gateway.

## How to run it

```bash
# yersinia, interactive or scripted
yersinia dhcp -attack 1 -interface eth0     # sends DISCOVER with random source MACs

# or the classic single-purpose tool
dhcpstarv -i eth0
```

Each forged DISCOVER uses a new MAC, so the server treats every one as a distinct client and
leases an address to each.

## What you see when it works

On the DHCP server, the lease table fills with addresses bound to MACs that do not exist:

```
gateway> cat /var/lib/dhcp/dhcpd.leases | grep -c "^lease"
  253        # the /24 pool is exhausted in seconds
```

A legitimate client booting now:

```
victim> dhclient eth0
  DHCPDISCOVER on eth0 ... no DHCPOFFERS received
  No working leases in persistent database - sleeping.
```

## The fingerprint it leaves

- **A burst of DHCP DISCOVER from many distinct source MACs on one switch port.** Real hosts
  on an access port are one or two MACs, not two hundred. The switch port sees an impossible
  number of MACs.
- **In the capture:** filter `dhcp` and watch the `dhcp.hw.mac_addr` field cycle through
  hundreds of unique values, all arriving on the same physical port.
- **Port-security violation counters climbing**, once the mitigation is in place.

## Why it works, in one line

The switch will forward frames from any source MAC on an access port by default. Capping the
number of MACs a port will learn — port security — is what turns two hundred forged clients
into a shut-down port.
