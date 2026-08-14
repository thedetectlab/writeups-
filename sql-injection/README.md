# ARP Spoofing: Anatomy of a MITM

*how a network ends up trusting the last reply it heard*

## The problem ARP was built to solve

Devices on a local network talk to each other using MAC addresses, but applications think in IP addresses. Something has to map one to the other. That something is ARP (Address Resolution Protocol) — and it was designed in 1982, for a trusted local network, with no authentication built in at all.

The process is simple by design:

```
Who has 192.168.1.1? Tell 192.168.1.50   (broadcast)
192.168.1.1 is at aa:bb:cc:dd:ee:ff       (reply)
```

The requesting device broadcasts a question to the whole subnet. Whoever owns that IP replies with its MAC address, and the requester caches the answer in its ARP table for future use. That cache is the entire trust model. There's no signature, no certificate, nothing tying a reply to the device it claims to come from.

## What "spoofing" actually means here

Most operating systems will accept an ARP reply *even if they never sent the matching request*. This is called a gratuitous ARP, and it exists for legitimate reasons — a device announcing a new IP, a failover system taking over an address. But nothing stops an attacker from sending one too.

An attacker on the same subnet sends two forged replies, one to each side of a conversation they want to intercept:

```
To the victim:   192.168.1.1 (the router) is at [attacker's MAC]
To the router:   192.168.1.50 (the victim) is at [attacker's MAC]
```

Both devices update their ARP cache with the lie. The victim now sends everything meant for the router to the attacker's machine instead, and the router sends everything meant for the victim to the attacker too. Neither side sees an error. Neither side gets a warning. The last ARP reply a device heard is the one it trusts, and there's no mechanism to question it.

## The MITM part

Once traffic from both directions is flowing through the attacker's machine, the attacker typically forwards it onward — router to victim, victim to router — so the connection keeps working and nobody notices a drop. This is the actual "man in the middle": every packet on that link passes through the attacker first, who can read it, log it, or modify it in transit before forwarding it along.

Anything unencrypted crossing that link — plaintext HTTP, unencrypted DNS, some IoT protocols — is now fully visible. Encrypted traffic (TLS) is protected from content inspection but not from the interception itself; the attacker still sees connection metadata and can attempt downgrade or certificate-based attacks on top of the MITM position.

## What it looks like in a packet capture

The signature is a MAC address flip-flopping for the same IP in a short window:

```
192.168.1.1 is at aa:bb:cc:dd:ee:ff
192.168.1.1 is at 11:22:33:44:55:66   ← 40 seconds later, same IP
192.168.1.1 is at aa:bb:cc:dd:ee:ff   ← flips back
```

A gateway address shouldn't move. Filtering on `arp` in Wireshark and watching the IP-to-MAC mapping for the default gateway across the capture is usually enough to spot an active attack — the legitimate router doesn't change its MAC, so any second claimant is either a failover event or spoofing, and the rate and pattern of replies tells you which. A flood of unsolicited gratuitous ARP replies claiming the gateway's IP, arriving faster than any real failover would need, is the tell.

## Why it's still possible in 2026

The fix — Dynamic ARP Inspection on managed switches, which validates ARP replies against a known IP-to-MAC-to-port binding table (usually built from DHCP snooping) — has existed for years and is standard on enterprise switching gear. It's simply not enabled by default on most consumer routers and small-business switches, and flat, unsegmented networks with no port security make the attack trivial once someone has a foothold on the LAN. The protocol hasn't changed since 1982; what's changed is which networks bothered to add the layer on top of it.

## Mitigation checklist

- Enable Dynamic ARP Inspection (or the vendor equivalent) on managed switches, backed by DHCP snooping.
- Use static ARP entries for high-value, low-change devices like gateways and servers, where operationally feasible.
- Segment the network — VLANs limit how much traffic a single compromised host on one segment can intercept.
- Prefer encrypted protocols end-to-end (HTTPS, DoH/DoT) so a successful MITM position yields metadata, not content.
- Monitor for ARP anomalies (rapid MAC changes for static IPs, unsolicited gratuitous ARP bursts) with an IDS tuned for it.
