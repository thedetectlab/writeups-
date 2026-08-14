# TCP 3-Way Handshake, Packet by Packet

*what Wireshark is really showing you in those first three lines*

## Why a handshake at all

TCP promises reliable, ordered delivery over a network that guarantees neither. Before either side sends real data, both need to agree on two things: that the other side is actually there and listening, and what starting sequence number each side will count from. The three-way handshake does exactly that — nothing more.

Open Wireshark on any TCP connection and the first three packets always look like this:

```
1  Client → Server   SYN                  Seq=x
2  Server → Client   SYN, ACK             Seq=y  Ack=x+1
3  Client → Server   ACK                  Seq=x+1  Ack=y+1
```

## Packet 1: SYN

The client picks a random initial sequence number — call it `x` — and sends a packet with the SYN flag set and no data.

```
Flags: SYN
Seq: x
```

This isn't a request in the HTTP sense. It's the client saying: *I want to open a connection, and I'm going to count my bytes starting from x.* The sequence number is randomized per connection specifically so it can't be predicted — a predictable ISN is what let old TCP sequence-prediction attacks hijack connections in the first place.

## Packet 2: SYN, ACK

The server responds with both flags set:

```
Flags: SYN, ACK
Seq: y
Ack: x+1
```

Two things are happening in this one packet. The ACK half acknowledges the client's SYN — `Ack: x+1` means "I received your sequence number x, next byte I expect is x+1." The SYN half is the server doing the same thing the client just did: picking its own random sequence number `y` and announcing it. This is why it's called a *three*-way handshake and not four — the acknowledgment and the server's own SYN are combined into a single packet instead of sent separately.

## Packet 3: ACK

The client acknowledges the server's SYN:

```
Flags: ACK
Seq: x+1
Ack: y+1
```

At this point both sides have sent a sequence number and had it acknowledged. The connection moves to the `ESTABLISHED` state on both ends, and this third packet can — and usually does — carry the first bytes of actual application data (an HTTP GET, a TLS ClientHello) riding along with the ACK.

## What actually breaks

**SYN floods.** An attacker sends packet 1 repeatedly with spoofed source addresses and never sends packet 3. The server allocates state for each half-open connection and waits for an ACK that's never coming. Enough of these exhaust the connection backlog queue, and legitimate clients can't get a SYN-ACK slot. SYN cookies are the standard mitigation — the server encodes the connection state into the sequence number itself instead of storing it, so no memory is committed until the real ACK arrives.

**RST instead of the expected packet.** If a firewall or a closed port rejects the SYN, you'll see an RST/ACK come back instead of a SYN/ACK — that's the fast, deliberate "nothing is listening here" response, distinct from a timeout, which usually means something is silently dropping the packet instead.

**Retransmitted SYNs in the capture.** If packet 1 shows up two or three times with increasing intervals before a SYN-ACK finally appears, that's the client's retransmission timer firing — the first attempt(s) were lost or the server was slow to respond, not a protocol issue.

## Reading it in Wireshark

Filter on `tcp.flags.syn==1` to isolate handshake attempts, or use `tcp.stream eq N` to follow one full connection in isolation once you've found the stream number for the conversation you care about. The `Seq` and `Ack` numbers Wireshark displays by default are *relative* to the start of the capture (starting at 0) rather than the raw values on the wire — useful for reading at a glance, but worth knowing when you're correlating against raw packet dumps from another tool.
