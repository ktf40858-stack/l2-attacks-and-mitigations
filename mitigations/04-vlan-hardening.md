# Mitigation 4 — VLAN hardening

**Defeats:** [VLAN hopping](../attacks/05-vlan-hopping.md), both variants

## The idea

Both hopping variants exploit defaults: ports that will auto-negotiate a trunk, and a native
VLAN shared with user traffic. Remove both defaults and both variants close.

## Configuration

```
! 1. Every user port is hard access. No negotiation, ever.
interface range GigabitEthernet0/1 - 2
 switchport mode access           ! not dynamic auto / dynamic desirable
 switchport access vlan 10
 switchport nonegotiate           ! stop sending/accepting DTP - kills switch-spoofing
 spanning-tree portfast
 spanning-tree bpduguard enable

! 2. Trunks are explicit, and their native VLAN is an unused parking VLAN,
!    NOT VLAN 1 and NOT any user VLAN. This kills double-tagging: the attacker
!    would have to be on VLAN 99, which carries no hosts.
interface GigabitEthernet0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20    ! prune everything else off the trunk

! 3. Shut and park every unused port, so an empty jack is not a door
interface range GigabitEthernet0/4 - 24
 switchport mode access
 switchport access vlan 99
 shutdown
```

## Why each line matters against the attack

| Line | Closes |
|---|---|
| `switchport mode access` + `nonegotiate` | Switch spoofing — the port will not become a trunk on request |
| `native vlan 99` (unused) | Double tagging — the outer tag must match the native VLAN to be stripped; no host lives on 99 to send it |
| `allowed vlan 10,20` | Limits blast radius — even a formed trunk carries only two VLANs, not all |
| unused ports shut + parked | Removes the free jack an attacker plugs into |

## Proving it works

Re-run the DTP attack from [attack 5A](../attacks/05-vlan-hopping.md):

```
S1# show interface gi0/2 switchport
  Administrative Mode: static access
  Operational Mode: static access        <-- stayed access; DTP got no response
  Negotiation of Trunking: Off
```

Yersinia's DTP negotiation now gets silence, and the double-tag frame dies because native
VLAN 99 carries no return path and no host to source the outer tag legitimately.

## The trade-off worth knowing

Pruning `allowed vlan` is the line people skip, and it is the one that limits damage when
something else fails. There is no downside to it except that a new VLAN has to be added to the
trunk explicitly — which is a feature, not a cost: VLANs should not appear on a trunk by
accident. The native-VLAN change must be made on **both ends** of every trunk simultaneously,
or the trunk mismatches; doing it on a live network takes a maintenance window.
