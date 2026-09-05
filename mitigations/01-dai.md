# Mitigation 1 — Dynamic ARP Inspection (DAI)

**Defeats:** [ARP spoofing / MITM](../attacks/01-arp-spoofing.md)

## The idea

DAI validates every ARP packet on an untrusted port against a table of known-good
IP-to-MAC bindings. An ARP reply whose IP/MAC pair does not match the binding is dropped and
logged. The binding table comes from **DHCP snooping** — the switch watched the DHCP exchange
and knows which MAC legitimately holds which IP — so DAI depends on DHCP snooping being on
first.

For statically addressed hosts, an ARP ACL supplies the binding manually.

## Configuration

```
! DHCP snooping must be enabled first - it builds the binding table DAI reads
ip dhcp snooping
ip dhcp snooping vlan 10,20

! turn DAI on for the user VLANs
ip arp inspection vlan 10,20

! the uplink toward the DHCP server and other switches is trusted - do not inspect it
interface GigabitEthernet0/3
 ip arp inspection trust
 ip dhcp snooping trust

! access ports stay untrusted (the default) and are rate-limited so an ARP flood
! cannot also become a CPU DoS against the inspection process
interface range GigabitEthernet0/1 - 2
 ip arp inspection limit rate 15

! for a statically addressed server, bind it explicitly
arp access-list STATIC-SERVERS
 permit ip host 192.168.20.20 mac host 00:0c:29:aa:bb:20
ip arp inspection filter STATIC-SERVERS vlan 20
```

## Proving it works

Re-run the ARP spoof from [attack 1](../attacks/01-arp-spoofing.md). Now:

```
S1# show ip arp inspection statistics vlan 10
  Vlan  Forwarded  Dropped  DHCP Drops  ...
  10    1044       892      892             <-- the forged replies are dropped
```

```
S1# show log | include ARP
  %SW_DAI-4-DHCP_SNOOPING_DENY: 1 Invalid ARPs (Res) on Gi0/2, vlan 10.
  ([00:0c:29:ff:ee:66/192.168.10.1/... ])       <-- attacker MAC claiming the gateway IP
```

The victim's ARP cache stays correct throughout. Capture `dai-blocked.pcap` shows the forged
replies never reaching the victim's port.

## The trade-off worth knowing

DAI rate-limiting can err-disable a trunk if the rate is set too low and a burst of legitimate
ARP occurs (a lot of hosts booting at once). Trusted ports are not rate-limited by default for
this reason; set access-port limits with headroom above normal ARP volume. This is exactly the
kind of tuning detail an interviewer probes — the feature is easy, living with it is the skill.
