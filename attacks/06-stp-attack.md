# Attack 6 — STP root bridge takeover

**ATT&CK:** T1498 Network Denial of Service (and MITM)
**Mitigation:** [BPDU Guard + Root Guard](../mitigations/05-stp-protection.md)

## What it does

Spanning Tree Protocol prevents Layer 2 loops by electing a root bridge — the switch with the
lowest bridge ID — and blocking redundant paths toward it. The election trusts whoever claims
the lowest bridge ID. An attacker sends BPDUs advertising a bridge priority of 0, wins the
election, and becomes the root. The topology reconverges around the attacker, so traffic
between switches is now steered through the attacker's port: a man-in-the-middle at the
switching fabric, or, by flapping the claim, a denial of service as the network reconverges
over and over.

## How to run it

```bash
yersinia stp -attack 4 -interface eth0     # claim to be root (priority 0)
```

Or with a crafted BPDU that just keeps lowering the priority to force continuous reconvergence.

## What you see when it works

The real switches now point their root port at the attacker:

```
S1# show spanning-tree vlan 10
  Root ID    Priority    0
             Address     00:0c:29:ff:ee:66      <-- the attacker's MAC is now root
             This bridge is not the root
```

Every reconvergence blackholes traffic for the tens of seconds STP takes to settle — and if
the attacker flaps, it never settles.

## The fingerprint it leaves

- **A topology change from an access port.** A user port should never source a superior BPDU.
  A host suddenly claiming to be the root bridge is the entire attack.
- **STP topology-change notifications (TCN) storming** — repeated reconvergence is visible as a
  flood of TCN BPDUs.
- **In the capture:** filter `stp` and look for a configuration BPDU with `stp.root.prio == 0`
  sourced from a host on an access port.
- **BPDU Guard err-disable log entries**, once the mitigation is on — the port shuts the instant
  it receives a BPDU it should never have seen.

## Why it works, in one line

STP accepts a superior BPDU from any port, including a user access port. BPDU Guard shuts any
access port that receives a BPDU at all, and Root Guard stops a downstream port from ever
becoming the path to a new root — so the attacker's claim shuts its own port instead of winning.
