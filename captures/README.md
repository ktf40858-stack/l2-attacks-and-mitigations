# Packet captures

Each attack has a before/after pair. Open them in Wireshark next to the matching attack page.

> The `.pcap` files are produced by running the lab. This directory documents the exact display
> filters so the captures are reproducible and so you can read your own. Committing large binary
> pcaps is optional — the filters below are what actually teach the material.

## Display filters per attack

| Attack | Filter | What to look for |
|---|---|---|
| ARP spoofing | `arp.opcode == 2` | one `arp.src.hw_mac` answering for several `arp.src.proto_ipv4`; Wireshark flags "duplicate use of IP" |
| DHCP starvation | `dhcp` | hundreds of unique `dhcp.hw.mac_addr`, all on one port |
| Rogue DHCP | `dhcp.option.dhcp == 2` | DHCP OFFER from two different server MACs |
| CAM overflow | `eth.src` (Statistics > Endpoints) | thousands of source MACs in seconds |
| VLAN hop (DTP) | `dtp` | DTP frames sourced from a host on an access port |
| VLAN hop (double tag) | `vlan && vlan.etype == 0x8100` | two stacked 802.1Q headers |
| STP takeover | `stp` | config BPDU with `stp.root.prio == 0` from an access port |

## Reading a before/after pair

The "before" capture shows the attack succeeding — forged replies reaching the victim, offers
from the rogue server, the CAM table leaking. The "after" capture, taken with the mitigation on,
shows the switch intervening: the forged frame never reaches the victim's port, the port
err-disables, the BPDU triggers a guard. The difference between the two files is the lesson.

## How to capture

```bash
# on the attacker, to prove the attack works (before)
tcpdump -i eth0 -w before.pcap <filter>

# on a SPAN/mirror port or the victim, to prove the mitigation works (after)
tcpdump -i eth0 -w after.pcap <filter>
```

On a physical switch, mirror the access port to a capture port:

```
monitor session 1 source interface Gi0/2
monitor session 1 destination interface Gi0/24
```
