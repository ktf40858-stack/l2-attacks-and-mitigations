# Layer 2 Attacks and Mitigations

Six Layer 2 attacks, run in a lab, each paired with the switch configuration that stops it
and the packet capture that proves it stopped. This is the repo that connects a CCNA to
security work: the same switching knowledge, turned toward the attack surface it creates.

> Every attack here is run against **my own lab switches and hosts**. Running these against a
> network you do not own is a crime in most jurisdictions. The point of the repo is the
> mitigation — the attack is only here to show what the mitigation defeats.

---

## The six

| # | Attack | Layer 2 mechanism abused | Mitigation | ATT&CK |
|---|---|---|---|---|
| 1 | [ARP spoofing / MITM](attacks/01-arp-spoofing.md) | ARP has no authentication | [Dynamic ARP Inspection](mitigations/01-dai.md) | T1557.002 |
| 2 | [DHCP starvation](attacks/02-dhcp-starvation.md) | DHCP pool exhaustion | [Port security + DHCP snooping](mitigations/02-dhcp-snooping.md) | T1498 |
| 3 | [Rogue DHCP server](attacks/03-rogue-dhcp.md) | No server authentication in DHCP | [DHCP snooping trust](mitigations/02-dhcp-snooping.md) | T1557 |
| 4 | [CAM table overflow](attacks/04-cam-overflow.md) | Finite MAC table, fail-open | [Port security](mitigations/03-port-security.md) | T1040 |
| 5 | [VLAN hopping](attacks/05-vlan-hopping.md) | DTP auto-trunking, native VLAN | [Explicit access ports, native VLAN change](mitigations/04-vlan-hardening.md) | T1599 |
| 6 | [STP manipulation](attacks/06-stp-attack.md) | STP trusts the lowest bridge ID | [BPDU Guard, Root Guard](mitigations/05-stp-protection.md) | T1498 |

Each attack page has the same four sections: **what it does**, **how to run it**, **what you
see when it works**, and **the fingerprint it leaves** — because detecting the attack matters
as much as blocking it, and a Tier 1 analyst needs both.

## Why Layer 2 is the soft underbelly

Almost all security effort goes into Layer 3 and above — firewalls, ACLs, segmentation. Layer
2 sits underneath all of it and was designed in an era that assumed the local segment was
trusted. ARP, DHCP, STP and DTP have no authentication by design. An attacker with a port in
the wall — a rogue access point, an unattended jack in a lobby, a compromised IP phone —
operates below the layer every firewall watches.

The defences exist and have for years. They are just frequently left off, because they are
switch features nobody enables until an assessment finds them missing. This repo enables them
and shows the difference on the wire.

## The lab

```
                         [ Switch S1 - Cisco IOS ]
                          Gi0/1        Gi0/2      Gi0/3
                            |            |          |
                        [ Victim ]  [ Attacker ] [ Gateway/DHCP ]
                        Win/Linux    Kali Linux    Linux + dnsmasq

VLAN 10  users     192.168.10.0/24
VLAN 20  servers   192.168.20.0/24
VLAN 99  native/parking (unused, all access ports default here)
```

Built in GNS3 with a Cisco IOSvL2 image and three VM endpoints; a physical Catalyst switch
with three laptops works identically. Full build in [`topology/lab-setup.md`](topology/lab-setup.md).

Tools on the attacker host: `arpspoof` / `ettercap`, `dhcpstarv` / `yersinia`, `macof`,
`dnsmasq` for the rogue DHCP server. All standard, all in Kali.

## How to read a capture

The `captures/` directory holds the before/after `.pcap` for each attack. Open them in
Wireshark alongside the attack page. The "after" capture is filtered to show the mitigation
doing its job — a DAI drop, a port-security violation, a BPDU Guard err-disable. Reading the
difference on the wire is the whole exercise.

Filters used are listed per attack in [`captures/README.md`](captures/README.md).

## Reproduce it

```bash
git clone https://github.com/ktf40858-stack/l2-attacks-and-mitigations
# 1. Build the lab                -> topology/lab-setup.md
# 2. Pick an attack               -> attacks/0X-*.md   (run it, confirm it works)
# 3. Apply the paired mitigation  -> mitigations/0X-*.md
# 4. Run the attack again         -> it now fails; capture the difference
```

## Author

Kodjo Apedoh — Network & Cloud Security · Arlington, VA
CCNA · Fortinet NSE · Palo Alto SASE & Cloud Security
[LinkedIn](https://www.linkedin.com/in/kodjo-apedoh-03030990/) · [Other labs](https://github.com/ktf40858-stack)

## License

MIT — see [LICENSE](LICENSE). Lab and educational use only.
