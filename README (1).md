<div align="center">

```
r o o t @ s n i f f e r : ~ / w r i t e u p s / o s i - m o d e l #
```

# 🧠 The OSI Model — How Data Actually Travels the Network

</div>

---

Every packet you capture in Wireshark passes through 7 layers before it means anything. Most people memorize the list once for an exam and never think about it again — which is a mistake, because the layers are exactly how you *diagnose* a problem: figure out which layer is broken, and you've cut the search space by 85%.

## 📚 The 7 Layers, Top to Bottom

```
7. APPLICATION    → HTTP, DNS, the stuff users actually see
6. PRESENTATION   → encryption, formatting (TLS lives here)
5. SESSION        → keeps the conversation open
4. TRANSPORT      → TCP/UDP, ports, reliability
3. NETWORK        → IP addresses, routing
2. DATA LINK      → MAC addresses, switches
1. PHYSICAL       → cables, signals, the actual wire
```

## 🔍 What Each Layer Actually Does

**Layer 1 — Physical**
The literal electrical signal, light pulse, or radio wave. No structure, no meaning — just bits moving through a medium. A bad cable or a dead NIC breaks everything above it.

**Layer 2 — Data Link**
Introduces the concept of a "frame" and the MAC address. This is where switches operate — they only care about MAC addresses, not IPs. ARP (Address Resolution Protocol) lives at the boundary of Layer 2/3, translating IP addresses to MAC addresses.

**Layer 3 — Network**
IP addresses and routing. This is where routers operate — they read the destination IP and decide which direction to forward the packet. No concept of "connection" yet, just best-effort delivery.

**Layer 4 — Transport**
TCP and UDP live here. This is where "ports" come from, and where reliability gets introduced — TCP's three-way handshake, sequence numbers, and retransmission all happen at this layer. UDP skips all of that for speed.

**Layer 5 — Session**
Manages whether a conversation stays open or gets torn down. In practice, this layer is thin — a lot of modern protocols fold session management into Layer 4 or 7 instead.

**Layer 6 — Presentation**
Formatting and encryption. TLS technically operates here — translating raw bytes into something both ends agree how to interpret (and encrypt/decrypt).

**Layer 7 — Application**
The layer users actually interact with: HTTP, DNS, SMTP, FTP. When you type a URL, Layer 7 is where that request is built.

## 🦈 Seeing It in Wireshark

Open any capture and expand a packet's details pane — the layers are laid out top to bottom, low to high:

```
Frame (physical capture metadata)
 └─ Ethernet II (Layer 2 — MAC addresses)
     └─ Internet Protocol (Layer 3 — IP addresses)
         └─ Transmission Control Protocol (Layer 4 — ports, flags)
             └─ Hypertext Transfer Protocol (Layer 7 — the actual request)
```

Notice Layers 5 and 6 usually don't get their own visible section — that's normal. Not every protocol stack uses them explicitly; TLS shows up as its own entry when present, folded near where Layer 6 would sit.

## 🧩 Why This Actually Matters for Troubleshooting

The OSI model isn't trivia — it's a diagnostic checklist:

| Symptom | Likely Layer |
|---|---|
| No link light, cable unplugged | 1 — Physical |
| Device on the network but can't reach anything, ARP failing | 2 — Data Link |
| Can ping by IP but nothing routes further | 3 — Network |
| Connection resets, port unreachable, timeouts | 4 — Transport |
| TLS handshake fails, certificate errors | 6 — Presentation |
| "The site is down" but ping/traceroute work fine | 7 — Application |

Start troubleshooting from the bottom up. There's no point debugging an HTTP 500 error if the machine can't even resolve DNS.

---

<div align="center">

```
TYPE      FIELD WRITEUP
STATUS    ACTIVE
```

Reference version of this topic lives in the [tool repos](https://github.com/thedetectlab/Featured-Work) — this is the explanation.

</div>
