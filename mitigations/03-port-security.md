# Mitigation 3 — Port security

**Defeats:** [CAM table overflow](../attacks/04-cam-overflow.md),
and blunts [DHCP starvation](../attacks/02-dhcp-starvation.md)

## The idea

Port security caps how many MAC addresses a port will learn, and decides what happens when the
cap is exceeded. An access port serves one host — maybe two, with an IP phone — so a limit of a
handful of MACs is generous. A `macof` flood presenting ten thousand MACs hits the cap on the
first extra one and the port reacts immediately.

## Configuration

```
interface range GigabitEthernet0/1 - 2
 switchport mode access
 switchport port-security
 switchport port-security maximum 2            ! host + one IP phone
 switchport port-security violation restrict   ! drop + log + count, do not hard-shut
 switchport port-security aging time 5
 switchport port-security aging type inactivity
 switchport port-security mac-address sticky   ! learn the current MAC and persist it
```

## The one decision that matters: violation mode

| Mode | On violation | Use when |
|---|---|---|
| `protect` | silently drops offending frames | almost never — no log means no visibility |
| `restrict` | drops, logs, increments the counter, sends an SNMP trap | **default choice** — you keep the port up and you find out |
| `shutdown` | err-disables the port entirely | high-security areas, or where a violation should mean a truck roll |

`shutdown` is the most secure and the most operationally painful: one visitor plugging a laptop
into a phone's passthrough port takes the port down until someone clears it. `restrict` catches
the attack, keeps legitimate users working, and still tells you. Choosing between them is a risk
conversation, not a technical one — and being able to have that conversation is the point.

## Proving it works

Re-run `macof` from [attack 4](../attacks/04-cam-overflow.md):

```
S1# show port-security interface gi0/2
  Port Security              : Enabled
  Port Status                : Secure-up
  Violation Mode             : Restrict
  Total MAC Addresses        : 2
  Security Violation Count   : 148213      <-- every forged MAC past the cap, counted and dropped
```

The CAM table stays clean, and the attacker never sees other ports' traffic:

```
S1# show mac address-table count
  Dynamic Address Count: 6        <-- not 8192; the flood went nowhere
```

## The trade-off worth knowing

Sticky MACs plus `shutdown` is a helpdesk generator: every legitimate device swap becomes a
ticket. In practice `restrict` with a sane maximum and inactivity aging is what survives contact
with real users. Port security is also per-port state — it does not follow a user who roams —
so it pairs with, rather than replaces, 802.1X where identity needs to move with the person.
