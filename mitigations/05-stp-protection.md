# Mitigation 5 — STP protection (BPDU Guard, Root Guard)

**Defeats:** [STP root bridge takeover](../attacks/06-stp-attack.md)

## The idea

Two complementary guards, applied at different places:

- **BPDU Guard** on access ports: a user port should never receive a BPDU. If it does,
  something is speaking STP that should not be — a rogue switch, or an attacker. BPDU Guard
  err-disables the port the instant a BPDU arrives. This is the primary defence.
- **Root Guard** on ports facing downstream switches: if a superior BPDU arrives that would
  make the neighbor the new root, the port is put in `root-inconsistent` state — blocked —
  until the superior BPDUs stop. This protects the *designed* root from being displaced by a
  switch that should never be root.

## Configuration

```
! BPDU Guard on every access port. A BPDU here = shut the port.
interface range GigabitEthernet0/1 - 2
 spanning-tree portfast
 spanning-tree bpduguard enable

! belt and braces: enable it globally for all portfast ports
spanning-tree portfast bpduguard default

! Root Guard toward downstream switches that must never become root
interface GigabitEthernet0/4
 spanning-tree guard root

! nail down the legitimate root so the election is not a free-for-all
spanning-tree vlan 10,20 priority 4096      ! this switch is root, deterministically

! auto-recover err-disabled ports after 5 min instead of a manual bounce
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

## Proving it works

Re-run the STP attack from [attack 6](../attacks/06-stp-attack.md):

```
S1# show log | include BPDU
  %SPANTREE-2-BLOCK_BPDUGUARD: Received BPDU on port Gi0/2 with BPDU Guard enabled.
  Disabling port.
  %PM-4-ERR_DISABLE: bpduguard error detected on Gi0/2, putting Gi0/2 in err-disable state
```

```
S1# show spanning-tree vlan 10
  This bridge is the root         <-- the attacker never displaced it; its port shut instead
```

The attacker's first BPDU shuts the attacker's own port. The topology never reconverges around
it, so there is no MITM and no reconvergence DoS.

## The trade-off worth knowing

BPDU Guard on a port where a legitimate switch later gets plugged in will shut that switch out —
which is correct, but surprises people who move equipment. That is why `errdisable recovery`
matters: without it, every tripped port is a manual `shutdown`/`no shutdown`, and on a large
network that is a real operational load. The deeper point for an interview: BPDU Guard protects
the *edge*, Root Guard protects the *core topology*, and they are not interchangeable — putting
Root Guard on an access port or BPDU Guard on a trunk to another legitimate switch are both
common misconfigurations that leave a gap.
