# Lab setup

## Diagram

```
                         [ Switch S1 - Cisco IOSvL2 ]
                    Gi0/1          Gi0/2          Gi0/3
                      |              |              |
                 [ Victim ]    [ Attacker ]   [ Gateway ]
                 192.168.10.20  Kali Linux    192.168.10.1
                 VLAN 10        VLAN 10        + dnsmasq (DHCP)
                                               VLAN 10 / trunk

  VLAN 10  users     192.168.10.0/24
  VLAN 20  servers   192.168.20.0/24   (one host, 192.168.20.20)
  VLAN 99  parking   unused - native VLAN on trunks, default for dead ports
```

## Two ways to build it

**GNS3 (no hardware).** A Cisco IOSvL2 image for the switch, three VM appliances for the hosts.
This is what the captures in this repo were taken from. IOSvL2 supports DTP, STP, DHCP snooping,
DAI and port security — everything the mitigations need.

**Physical.** Any Catalyst 2960/3560-class switch and three laptops. Identical commands. If you
have the hardware, use it — the err-disable and recovery behaviour feels more real when a port
light actually goes amber.

## Base switch configuration (pre-hardening)

The lab starts **deliberately insecure** so the attacks work. This is the "before":

```
hostname S1
!
vlan 10
 name USERS
vlan 20
 name SERVERS
vlan 99
 name PARKING
!
interface range GigabitEthernet0/1 - 2
 switchport access vlan 10
 ! note: no 'switchport mode access', no port-security - default, vulnerable
!
interface GigabitEthernet0/3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 ! note: native VLAN is 1 by default, DTP is on - vulnerable to hopping
!
! no ip dhcp snooping, no ip arp inspection - vulnerable to rogue DHCP + ARP spoof
end
```

## Endpoint setup

**Gateway / DHCP server** (Linux):

```bash
# legitimate DHCP for VLAN 10
apt install -y isc-dhcp-server
cat > /etc/dhcp/dhcpd.conf <<CONF
subnet 192.168.10.0 netmask 255.255.255.0 {
  range 192.168.10.100 192.168.10.200;
  option routers 192.168.10.1;
  option domain-name-servers 192.168.10.1;
}
CONF
systemctl enable --now isc-dhcp-server
```

**Attacker** (Kali): everything needed is preinstalled — `dsniff` (arpspoof), `yersinia`,
`dsniff`/`macof`, `ettercap`, `dnsmasq`, `scapy`. Enable forwarding when doing MITM so the
victim stays online:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward
```

**Victim** (Windows or Linux): a plain DHCP client, nothing special. This is the host whose
experience you watch to confirm each attack and each fix.

## Order to work through it

Do the attacks against the insecure base config first and confirm each one works — a mitigation
you cannot see defeating a live attack proves nothing. Then apply the mitigations one at a time,
re-run the matching attack, and capture the difference. That before/after pair is the whole
value of the repo.

## Safety and legality

This lab is isolated. Nothing here touches a network you do not own. `macof`, `yersinia` and a
rogue DHCP server are genuinely disruptive — running them on a production or shared network is
both a crime and a good way to take down a building. Keep it on host-only / internal networking
in the hypervisor, or on an air-gapped physical switch.
