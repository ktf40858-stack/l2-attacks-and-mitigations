# Mitigation 2 — DHCP snooping

**Defeats:** [DHCP starvation](../attacks/02-dhcp-starvation.md),
[rogue DHCP server](../attacks/03-rogue-dhcp.md)

## The idea

DHCP snooping makes the switch a referee for DHCP. Every port is **untrusted** by default:
untrusted ports may send client messages (DISCOVER, REQUEST) but any **server** message
(OFFER, ACK) arriving on them is dropped. Only the one port toward the real DHCP server is
marked **trusted**. A rogue server plugged into a user port therefore has its OFFERs dropped
before they leave that port.

As a side effect the switch builds the **DHCP snooping binding table** — which MAC got which
IP on which port — and that table is what DAI (mitigation 1) and IP Source Guard read.

## Configuration

```
ip dhcp snooping
ip dhcp snooping vlan 10,20
no ip dhcp snooping information option    ! unless the upstream server expects option 82

! the ONLY trusted port: the uplink toward the real DHCP server
interface GigabitEthernet0/3
 ip dhcp snooping trust

! access ports: untrusted, and rate-limited to blunt starvation
interface range GigabitEthernet0/1 - 2
 ip dhcp snooping limit rate 10
```

The rate limit is what stops starvation: a port sending more than 10 DHCP packets per second
is err-disabled, so a `dhcpstarv` flood shuts its own port.

## Proving it works

**Against the rogue server** ([attack 3](../attacks/03-rogue-dhcp.md)):

```
S1# show log | include DHCP_SNOOPING
  %DHCP_SNOOPING-5-DHCP_SNOOPING_UNTRUSTED_PORT: received a DHCP OFFER
  message on untrusted port Gi0/2.        <-- the rogue's offer, dropped
```

The victim now only ever receives the legitimate offer; `ip route` shows the real gateway.

**Against starvation** ([attack 2](../attacks/02-dhcp-starvation.md)):

```
S1# show interface gi0/2 status
  Gi0/2   err-disabled    ...     <-- the flood tripped the rate limit and shut the port
```

The binding table, now populated, is visible with:

```
S1# show ip dhcp snooping binding
  MacAddress          IpAddress       Lease(sec)  Type          VLAN  Interface
  00:0C:29:11:22:20   192.168.10.20   86400       dhcp-snooping 10    GigabitEthernet0/1
```

## The trade-off worth knowing

The trusted-port list is the whole security boundary, and it is easy to get wrong. Trust the
wrong port and a rogue server there is fully authorised; forget to trust the real uplink and
the entire VLAN loses DHCP. On a network with DHCP relay, the port toward the relay (not the
server) is the trusted one. This is the setting an assessment checks first.
