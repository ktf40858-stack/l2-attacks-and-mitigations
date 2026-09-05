# Attack 5 — VLAN hopping

**ATT&CK:** T1599 Network Boundary Bridging
**Mitigation:** [Explicit access ports + native VLAN hardening](../mitigations/04-vlan-hardening.md)

VLANs are a segmentation boundary. VLAN hopping crosses it from an access port, which defeats
the assumption that a host in the users VLAN cannot reach the servers VLAN at Layer 2. There
are two ways in.

## Variant A — Switch spoofing (DTP)

**What it does.** Cisco's Dynamic Trunking Protocol negotiates whether a link becomes a trunk.
On default configuration a port is set to `dynamic auto` or `dynamic desirable`, meaning it
will *form a trunk if asked*. An attacker sends a DTP frame pretending to be a switch, the port
becomes a trunk, and a trunk carries **every VLAN**. The attacker now reaches all of them.

```bash
yersinia dtp -attack 1 -interface eth0     # negotiate a trunk
# once trunked, tag frames for the target VLAN with 8021q subinterfaces
vconfig add eth0 20
ifconfig eth0.20 192.168.20.66 netmask 255.255.255.0
```

**What you see.** The port transitions to trunk, and the attacker can now address hosts in
VLAN 20 from an access jack that was supposed to be VLAN 10 only:

```
S1# show interface gi0/2 switchport
  Administrative Mode: dynamic auto
  Operational Mode: trunk           <-- became a trunk on request
```

## Variant B — Double tagging (native VLAN)

**What it does.** A trunk's native VLAN is sent untagged. An attacker on the native VLAN sends
a frame with **two** 802.1Q tags: outer tag = native VLAN, inner tag = target VLAN. The first
switch strips the outer (native) tag as it forwards over the trunk, and the second switch reads
the now-exposed inner tag and delivers the frame into the target VLAN. It is one-way (no reply
path) but enough to inject traffic across the boundary.

```bash
# scapy: outer tag = native (99), inner tag = target (20)
python3 -c "from scapy.all import *; \
  sendp(Ether()/Dot1Q(vlan=99)/Dot1Q(vlan=20)/IP(dst='192.168.20.20')/ICMP())"
```

## The fingerprint it leaves

- **DTP frames arriving on an access port.** A user port has no reason to speak DTP. Yersinia's
  DTP negotiation is visible in a capture as `dtp` frames from a host MAC.
- **A port unexpectedly in trunk mode** — `show interface switchport` is the check.
- **Double-tagged frames** — in Wireshark, an 802.1Q header immediately followed by a second
  802.1Q header (`vlan and vlan.etype == 0x8100`) is never legitimate on an access port.

## Why it works, in one line

Ports that auto-negotiate trunking, and a native VLAN shared with user traffic, are the two
defaults that make hopping possible. Setting every user port to a hard `switchport mode access`
with DTP off, and moving the native VLAN to an unused parking VLAN, closes both variants.
