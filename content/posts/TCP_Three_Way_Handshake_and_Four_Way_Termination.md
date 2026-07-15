+++
title = 'TCP_Three_Way_Handshake_and_Four_Way_Termination'
date = 2026-07-16T00:00:00+08:00
draft = false
categories = ['Network']
tags = ['TCP', 'Network']
+++

> Study notes rewritten in English. 

## TCP basics

### What is TCP?

TCP is a **connection-oriented**, **reliable**, **byte-stream** transport-layer
protocol.

- **Connection-oriented** – strictly one-to-one. Unlike UDP, one host cannot
  broadcast to many hosts over a single TCP connection.
- **Reliable** – no matter what happens on the link, TCP guarantees a segment
  eventually reaches the receiver.
- **Byte stream** – the OS may split a user message into several TCP segments.
  Segments are ordered: if the previous segment hasn't arrived, a later one that
  did arrive cannot be handed up yet; duplicate segments are dropped.

IP is **unreliable** — it doesn't guarantee delivery, ordering, or integrity. If
you need reliability, the transport-layer TCP has to provide it.

### TCP header — the fields that matter

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-------------------------------+-------------------------------+
|          Source Port          |       Destination Port        |
+-------------------------------+-------------------------------+
|                        Sequence Number                        |
+---------------------------------------------------------------+
|                    Acknowledgment Number                      |
+-------+-------+-+-+-+-+-+-+------------------------------------+
| Offset| Rsvd  |U|A|P|R|S|F|             Window Size           |
+-------+-------+-+-+-+-+-+-+------------------------------------+
|           Checksum            |         Urgent Pointer        |
+-------------------------------+-------------------------------+
|                    Options (0 - 40 bytes)  ...                |
+---------------------------------------------------------------+
```

- **Sequence number** – a random ISN at connection time; each byte sent bumps it
  by the number of data bytes. Solves out-of-order delivery.
- **Acknowledgment number** – the next sequence number the sender *expects*;
  everything before it is considered received. Solves packet loss.
- **Control bits**:
  - `ACK` – acknowledgment field is valid (set on every segment after the initial SYN).
  - `RST` – abnormal connection, force a reset.
  - `SYN` – request to connect; carries the initial sequence number.
  - `FIN` – no more data to send; request to close.

### What is a connection? (RFC 793)

> The combination of sockets, sequence numbers, and window sizes … is called a
> connection.

So a TCP connection is a shared agreement on three things: **socket** (IP + port),
**sequence numbers**, and **window size** (flow control).

A connection is uniquely identified by a **4-tuple**:

```
( source IP, source port, destination IP, destination port )
```

The IPs (32-bit) live in the IP header; the ports (16-bit) live in the TCP header
and tell TCP which process to deliver to.

### How many connections can a server hold?

Theoretically, a listening server's max is `client_IP × client_port` ≈
`2^32 × 2^16 = 2^48`. In practice it's far lower, bounded by:

- **File descriptors** – every connection is a file; exhausting them gives
  `Too many open files` (system / user / process limits).
- **Memory** – each connection costs memory; exhausting it triggers OOM.

### TCP vs UDP

| | TCP | UDP |
|---|---|---|
| Connection | connection-oriented | connectionless |
| Endpoints | one-to-one | 1-to-1, 1-to-many, many-to-many |
| Reliability | reliable, ordered, no dup | best-effort |
| Flow / congestion control | yes | no |
| Header size | 20 bytes (no options) | 8 bytes, fixed |
| Boundaries | stream, no boundaries | per-datagram boundaries |
| Fragmentation | at transport layer, by **MSS** | at IP layer, by **MTU** |

**Use cases.** TCP suits reliable, connection-oriented transfers — **FTP** and
**HTTP/HTTPS**. UDP suits connectionless, low-overhead traffic — small-payload
protocols like **DNS** and **SNMP**, real-time **audio/video**, and **broadcast**.
(UDP can still be made reliable in userspace — e.g. QUIC.)

**Why does UDP have no "header length" field, while TCP does?** TCP has a
variable-length **Options** field, so it needs a header-length (data-offset) field
to know where the data begins. UDP's header is a fixed 8 bytes, so no such field
is required.

**Why does UDP have a "packet length" field, while TCP does not?** TCP derives its
payload length arithmetically — `TCP data = IP total length - IP header length - TCP header length`
— and all three are known. UDP also rides on IP and could compute the same way,
which makes its length field look redundant. Two plausible explanations:

- **4-byte alignment** — hardware handles headers more easily when their length is
  a multiple of 4 bytes. Without the 2-byte length field the UDP header would be 6
  bytes (source port + dest port + checksum), not a multiple of 4; the length field
  pads it to 8.
- **History** — UDP may originally have run over network-layer protocols that
  didn't expose their own length, so it carried its own field to allow the
  calculation.

> **Can TCP and UDP share the same port?** Yes. They are independent kernel
> modules; the IP header's protocol field selects TCP vs UDP, then the port
> selects the process. TCP:80 and UDP:80 do not conflict.

## Connection setup — the three-way handshake

```
  Client                                      Server
  CLOSED                                      LISTEN
    |                                              |
    |------------ SYN  seq=client_isn ------------>|   client: SYN_SENT
    |                                              |
    |<---------- SYN+ACK  seq=server_isn ----------|   server: SYN_RCVD
    |      ack=client_isn+1                        |
    |----------- ACK  ack=server_isn+1 ----------->|   client & server: ESTABLISHED
    |                                              |
```

1. Both start in `CLOSE`; the server does a passive open and sits in `LISTEN`.
2. The client picks a random `client_isn`, sends a **SYN**, moves to `SYN_SENT`.
3. The server picks a random `server_isn`, replies **SYN+ACK**
   (`ack = client_isn + 1`), moves to `SYN_RCVD`.
4. The client replies **ACK** (`ack = server_isn + 1`) and reaches `ESTABLISHED`;
   the server reaches `ESTABLISHED` on receipt.

The **third** ACK may carry data; the first two segments may not. On Linux, check
states with `netstat -napt`.

### Why three, not two or four?

The stock answer ("to confirm both sides can send and receive") is incomplete.
The real reasons:

1. **Prevent old duplicate (history) connections — the main reason.** RFC 793:
   *"the principal reason for the three-way handshake is to prevent old duplicate
   connection initiations from causing confusion."* If a stale SYN arrives first,
   the server replies SYN+ACK; the client sees an unexpected `ack` and answers
   `RST`, which tears down the half-open history connection before any data flows.
   A two-way handshake gives the server no intermediate state to reject a history
   connection, so it would establish and send data uselessly — wasting resources.
2. **Synchronize both initial sequence numbers.** Each side's ISN must be
   acknowledged by the other. That's inherently a 4-message exchange, but the
   server's ACK and SYN are folded into one segment → three.
3. **Avoid resource waste.** Without the third handshake the server can't tell
   whether the client got its SYN+ACK, so every repeated SYN would spawn a
   redundant connection.

### Why is the ISN randomized each time?

- **Prevent history packets** from a previous connection (same 4-tuple) from
  landing inside the new connection's receive window and being wrongly accepted.
- **Security** — makes it harder to forge segments with a guessed sequence number.

RFC 793 ISN: `ISN = M + F(localhost, localport, remotehost, remoteport)`, where
`M` is a timer incremented every 4 µs (wraps in ~4.55 h) and `F` is a keyed hash
(e.g. MD5) so it can't be trivially predicted.

### MSS vs MTU — why TCP segments itself

- **MTU** – largest link-layer frame payload, typically **1500 bytes** on Ethernet.
- **MSS** – largest TCP *data* per segment, i.e. MTU minus IP and TCP headers.

If TCP handed an oversized segment to IP and let **IP fragment** it, a single lost
fragment would force retransmission of the **entire** IP datagram (IP has no
retransmission of its own; it relies on TCP timeouts). By segmenting at the TCP
layer to fit the **MSS**, the resulting IP packets never exceed the MTU, so IP
never fragments — and a lost segment only retransmits that **one MSS**, not
everything.

### What if a handshake segment is lost?

- **SYN (1st) lost** – client retransmits SYN, controlled by `tcp_syn_retries`
  (default 5). Timeouts double each time: 1, 2, 4, 8, 16 s (+32 s wait) ≈ 63 s.
- **SYN+ACK (2nd) lost** – client retransmits SYN (`tcp_syn_retries`) *and* server
  retransmits SYN+ACK (`tcp_synack_retries`, default 5).
- **ACK (3rd) lost** – ACKs are never retransmitted themselves; the server
  retransmits SYN+ACK until it gets the third handshake or hits the retry limit.

## SYN flood attack

An attacker floods spoofed SYNs from random IPs. Each makes the server allocate a
half-open entry and reply SYN+ACK that never gets a final ACK, filling the
**half-connection (SYN) queue** so legitimate clients can't connect.

```
                 SYN queue                    Accept queue
                (half-open)                   (established)
  SYN --> [ create half-conn ] --3rd ACK--> [ full conn ] --accept()--> app
          server: SYN_RCVD                   server: ESTABLISHED
```

Both queues are bounded; overflow means dropped segments. Mitigations:

1. **Enlarge `netdev_max_backlog`** (NIC backlog when the kernel is slower than the wire).
2. **Enlarge the SYN queue** (`tcp_max_syn_backlog`, `listen()` backlog, `somaxconn`).
3. **Enable `tcp_syncookies`** — encode connection state into the SYN+ACK sequence
   number so connections succeed *without* using the SYN queue (`= 1` is right for
   defense: use cookies only when the queue is full).
4. **Reduce SYN+ACK retransmissions** (`tcp_synack_retries`) so half-open entries
   drain faster.

## Connection teardown — the four-way termination

```
  Client (active)                   Server (passive)
  ESTABLISHED                            ESTABLISHED
    |                                              |
    |---------------- FIN  seq=u ----------------->|   client: FIN_WAIT_1
    |<--------------- ACK  ack=u+1 ----------------|   server: CLOSE_WAIT
    |                                              |   client: FIN_WAIT_2
    |      ...server sends any leftover data...    |
    |<---------------- FIN  seq=v -----------------|   server: LAST_ACK
    |--------------- ACK  ack=v+1 ---------------->|   client: TIME_WAIT
    |      (client waits 2MSL)                     |   server: CLOSE
    |                                              |
  CLOSED                                      CLOSED
```

1. Client `close()` → **FIN**, enters `FIN_WAIT_1`.
2. Server **ACK**s, enters `CLOSE_WAIT`; client enters `FIN_WAIT_2`.
3. Server finishes sending, `close()` → **FIN**, enters `LAST_ACK`.
4. Client **ACK**s, enters `TIME_WAIT`; server reaches `CLOSE` on receipt. After
   `2MSL` the client also reaches `CLOSE`.

Only the **side that closes actively** gets a `TIME_WAIT` state.

**Why four?** The passive side's `ACK` (for the peer's FIN) and its own `FIN` are
usually **separate** — it may still have data to send before it agrees to close —
so the two can't be merged the way SYN+ACK is during setup. (When the passive
side has nothing left to send, four can collapse to three.)

### What if a termination segment is lost?

FIN retransmission is bounded by `tcp_orphan_retries`; an `ACK` is never
retransmitted on its own (the peer just resends its FIN).

- **FIN (1st) lost** – the active closer stays in `FIN_WAIT_1`, retransmits the FIN
  up to `tcp_orphan_retries` times, then gives up and goes to `CLOSE`.
- **ACK (2nd) lost** – ACKs aren't retransmitted, so the active closer keeps
  resending its FIN until the ACK arrives (or the retry limit is hit). Once it gets
  the ACK it moves to `FIN_WAIT_2` to wait for the server's FIN. For a `close()`d
  connection, `FIN_WAIT_2` lasts at most `tcp_fin_timeout` (default 60 s); for a
  `shutdown()` that closed only the send direction, it can persist indefinitely.
- **FIN (3rd) lost** – the passive closer stays in `LAST_ACK` and retransmits its
  FIN (again `tcp_orphan_retries`) until the final ACK arrives.
- **ACK (4th) lost** – the active closer is already in `TIME_WAIT`; the passive
  closer (still in `LAST_ACK`) retransmits its FIN. A FIN that arrives during
  `TIME_WAIT` resets the 2MSL timer.

## TIME_WAIT

### Why wait 2×MSL?

**MSL** (Maximum Segment Lifetime) is the longest a segment can live on the
network (Linux: 30 s; a related bound is the IP `TTL`, in hops). `2MSL` covers one
full round trip: if the final ACK is lost within one MSL, the peer's retransmitted
FIN arrives within the second MSL and the `TIME_WAIT` connection can still respond.
It tolerates one lost packet — 4×/8× MSL would only guard against back-to-back
losses, which are vanishingly rare. On Linux `2MSL` is a fixed 60 s
(`TCP_TIMEWAIT_LEN`). Receiving another FIN resets the timer.

### Why is TIME_WAIT needed?

1. **Prevent stale data from a previous connection** (same 4-tuple) from being
   accepted by a new one. `2MSL` is enough for both directions' packets to die out.
2. **Ensure the passive side closes cleanly.** If the final ACK is lost and the
   client had already closed, the retransmitted FIN would hit a closed client and
   get an `RST` — an ungraceful `Connection reset by peer`. `TIME_WAIT` keeps the
   client around long enough to re-ACK.

### Harm of too many, and how to tune

Too many `TIME_WAIT` sockets consume **system resources** (fds, memory, CPU) and,
on the client side, **ports** (`ip_local_port_range`, ~28k by default) — once
exhausted the client can't open new connections *to the same dst IP:port* (other
destinations still work, since the 4-tuple differs).

Tuning (each with trade-offs):

- `net.ipv4.tcp_tw_reuse = 1` (+ `tcp_timestamps`) — reuse TIME_WAIT sockets for
  new **outgoing** connections (client side only).
- `net.ipv4.tcp_max_tw_buckets` — hard cap; excess TIME_WAIT sockets are reset
  (blunt).
- `SO_LINGER` with `l_onoff=1, l_linger=0` — send `RST` on close, skipping
  four-way + TIME_WAIT entirely (dangerous, not recommended).

> *"TIME_WAIT is our friend"* (UNIX Network Programming) — don't try to eliminate
> it; understand it. A server that wants to avoid TIME_WAIT should simply let the
> **client** close first.

### Why do servers accumulate lots of TIME_WAIT?

`TIME_WAIT` appears on the **active closer**, so lots of it on a server means the
server is closing many connections — usually an **HTTP keep-alive** issue:

1. Keep-alive off (either side sends `Connection: close`) → server closes each
   response.
2. Keep-alive **timeout** (e.g. nginx `keepalive_timeout`) fires on idle connections.
3. Keep-alive **request cap** reached (nginx `keepalive_requests`, default 100) →
   server closes the connection.

Fix: keep keep-alive on for both sides, and raise the timeout / request cap for
high-QPS services.

### And lots of CLOSE_WAIT?

`CLOSE_WAIT` is the **passive closer** waiting for the app to call `close()`. Piles
of it almost always mean an application bug — the code never calls `close()` (e.g.
the socket was never registered with epoll, `accept()` was never called, or a
handler threw before closing).

## When an endpoint just vanishes

- **Client disappears (power off / crash of the machine)** — with no traffic, the
  server can't tell and the connection sits in `ESTABLISHED` forever. TCP
  **keepalive** guards against this: after `tcp_keepalive_time` (default 2 h) idle,
  it probes every `tcp_keepalive_intvl` (75 s); after `tcp_keepalive_probes` (9)
  unanswered probes the connection is declared dead. (~2 h 11 m total — apps often
  add a faster application-layer heartbeat instead.)
- **Server *process* crashes** — the kernel owns the connection, so it sends the
  FIN and completes the four-way termination on the process's behalf. (`kill -9`
  still produces a clean teardown.)

## Socket programming (how the API maps to the handshake)

```
   Server                             Client
   socket()                          socket()
   bind()      (assign IP:port)
   listen()    (start listening)
   accept()  ...blocks...
      :         <----- SYN -------   connect()    <- 3-way handshake begins
      :         ---- SYN+ACK --->
      :         <----- ACK ------                 connect() returns
   accept() returns                               (connect after 2nd handshake;
   = a NEW "connected socket"                       accept after the 3rd)
      |
   read()  <--------------- write()  data
   read() == EOF  <-------- close()  (client FIN)
   close()
```

- The **listening socket** and the **connected socket** are two different sockets —
  `accept()` returns a fresh fd used only for that one connection's data.
- **`backlog`** (in `listen(fd, backlog)`) — since Linux 2.2 it sizes the **accept
  queue** of completed connections, capped by `somaxconn`, i.e.
  `accept queue = min(backlog, somaxconn)`. (The half-open **SYN queue** is the one
  a SYN flood fills.)
- **When do the calls return?** `connect()` returns after the **second** handshake
  (the client received SYN+ACK); `accept()` returns after the **third** (the
  connection is `ESTABLISHED` and sitting in the accept queue).
- **Can you connect without `accept()`?** Yes — `accept()` takes no part in the
  handshake; the kernel completes it and queues the socket, and `accept()` merely
  pops it off afterwards.
- **Without `listen()`?** Also yes — two peers can **simultaneously open** (each
  SYNs the other), and a host can even connect to **itself**; neither needs a
  listening server.

## Summary

- A TCP connection = **socket + sequence numbers + window**, uniquely identified
  by the **4-tuple**.
- **Three-way handshake** exists mainly to reject old duplicate connections, and
  also to sync both ISNs and avoid wasted resources.
- **Four-way termination** needs four because the passive side's ACK and FIN are
  usually separate.
- **TIME_WAIT** (2MSL, active closer only) flushes stale data and lets the passive
  side close gracefully — tune it, don't bypass it.
- **SYN cookies** defend against SYN floods; **keepalive** detects dead peers.

---
