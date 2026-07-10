+++
title = 'How_Linux_Sends_and_Receives_Network_Packets'
date = 2026-07-11T07:00:00+08:00
draft = false
categories = ['Network']
tags = ['Linux', 'Network']
+++

> Study notes rewritten in my own words

## Network models

To let all kinds of devices talk to each other over a network, the ISO defined
the **OSI reference model** with 7 layers. Each layer has a single job:

- **Application** – provides a uniform interface to programs
- **Presentation** – converts data into a format another system understands
- **Session** – establishes, manages and terminates sessions
- **Transport** – end-to-end data transfer
- **Network** – routing, forwarding, fragmentation
- **Data link** – framing, error detection, MAC addressing
- **Physical** – transmits frames over the physical medium

OSI is mostly a *conceptual* model — it never shipped a concrete implementation.
What we actually use is the simpler **TCP/IP model** with 4 layers, and the Linux
stack is built to match it:

- **Application** – HTTP, DNS, FTP, …
- **Transport** – end-to-end delivery (TCP, UDP)
- **Network** – packet encapsulation, fragmentation, routing (IP, ICMP)
- **Link (network interface)** – framing, MAC addressing, error checking, and
  pushing frames out through the NIC

```
       OSI model                 TCP/IP model
  ┌──────────────────┐      ┌──────────────────┐
  │   Application    │      │                  │
  ├──────────────────┤      │                  │
  │   Presentation   │      │   Application    │
  ├──────────────────┤      │                  │
  │   Session        │      │                  │
  ├──────────────────┤      ├──────────────────┤
  │   Transport      │      │   Transport      │
  ├──────────────────┤      ├──────────────────┤
  │   Network        │      │   Network        │
  ├──────────────────┤      ├──────────────────┤
  │   Data link      │      │   Link (network  │
  ├──────────────────┤      │   interface)     │
  │   Physical       │      │                  │
  └──────────────────┘      └──────────────────┘
```

> When people say **L7** vs **L4** load balancing, they are speaking in OSI terms:
> L7 is the application layer, L4 is the transport layer.

## The Linux network protocol stack

A nice analogy: your body is the application data, your base layer is the TCP
header, your coat is the IP header, and your hat and shoes are the frame header
and frame tail. To go outside you put them on in order — that is exactly how a
packet is wrapped as it travels **down** the stack.

Each layer prepends (and the link layer also appends) its own header:

```
Application:                           [ App data ]
Transport:                   [ TCP hdr | App data ]
Network:            [ IP hdr | TCP hdr | App data ]
Link:   [ Frame hdr | IP hdr | TCP hdr | App data | Frame tail ]
```

Every added header/tail grows the packet, but a physical link cannot carry a
packet of arbitrary size. Ethernet fixes the **MTU (Maximum Transmission Unit)**
at **1500 bytes** — the largest IP packet a single transmission may carry. When
a packet exceeds the MTU it is **fragmented** at the network layer. A smaller MTU
means more fragments and worse throughput; a larger MTU means fewer fragments and
better throughput.

Knowing the model and the encapsulation, the Linux stack looks roughly like this:

```
              Application  (user space)
                   │   system call
  ─────────────────┼──────────────────────────  kernel
                 Socket
          ┌────────┼────────┐
         TCP      UDP      ICMP
          └────────┼────────┘
                  IP
                   │
                  MAC
                   │
              NIC driver
                   │
              NIC  (hardware)
```

- Applications talk to the **Socket** layer through system calls.
- Below the socket sit the transport, network and link layers.
- At the very bottom is the **NIC driver** and the physical **NIC**.

## Receiving a packet

When a NIC receives a frame it uses **DMA** to write the frame straight into a
pre-agreed memory region — the **Ring Buffer**, a circular queue — and then it
has to tell the OS the packet has arrived.

The simplest signal is an **interrupt** per packet. That breaks down under load:
at high packet rates the CPU is buried in interrupts, constantly dropping what it
was doing to service them, which starves everything else.

To fix this, Linux 2.6 introduced **NAPI**, a hybrid of *interrupt* + *polling*:
an interrupt wakes the receive routine, then the kernel **polls** for more data
instead of taking one interrupt per packet.

The flow:

1. The NIC DMAs the frame into the Ring Buffer and raises a **hardware interrupt**.
2. The registered **hard IRQ handler** runs and deliberately does very little:
   - It **temporarily masks** further interrupts — "I know there's data, just keep
     writing to memory, don't interrupt me again" — which avoids interrupt storms.
   - It raises a **soft interrupt (softirq)** and restores the masked interrupts.
3. The kernel thread **`ksoftirqd`** handles the softirq. It pulls a frame from the
   Ring Buffer, wraps it in an **`sk_buff`**, and hands it to the protocol stack.

Up the stack it goes:

- **Link layer** – checks the frame is valid (drop if not), reads the upper-layer
  protocol type (IPv4 / IPv6), strips the frame header and tail, passes it up.
- **Network layer** – takes the IP packet, decides its fate (deliver locally vs
  forward). For local delivery it reads the transport protocol (TCP/UDP) from the
  IP header, strips the IP header, and passes it up.
- **Transport layer** – strips the TCP/UDP header, uses the 4-tuple
  *(src IP, src port, dst IP, dst port)* to find the matching **Socket**, and drops
  the payload into that socket's receive buffer.
- **Application** – calls a socket read; the kernel **copies** the data from the
  socket receive buffer into the app's buffer and wakes the process.

## Sending a packet

Sending is the receive flow in reverse.

1. The app calls a socket send. Being a system call, this traps from user space
   into the kernel's **Socket** layer. The kernel allocates a kernel-side
   **`sk_buff`**, **copies** the user data into it, and queues it on the send buffer.
2. The stack takes the `sk_buff` off the send buffer and processes it top-down.
3. For **TCP**, the kernel first **clones** the `sk_buff`. TCP supports
   retransmission, so the original must survive until the peer's **ACK** arrives —
   only the clone is handed down and freed after transmission.
4. **Transport layer** fills in the TCP header.
5. **Network layer** selects a route (next-hop IP), fills the IP header, runs
   **netfilter**, and fragments anything larger than the MTU.
6. **Link layer** resolves the next-hop MAC via **ARP**, adds the frame header and
   tail, and places the `sk_buff` on the NIC's transmit queue.
7. A **softirq** notifies the driver, which reads the `sk_buff` from the queue,
   hangs it on the **Ring Buffer**, DMA-maps it into NIC-accessible memory, and
   triggers the real transmit.
8. After transmission the NIC raises a **hardware interrupt** to free memory
   (the `sk_buff` clone and Ring Buffer entries).
9. Finally, when the **ACK** for this segment arrives, the transport layer frees
   the **original** `sk_buff`.

### Why one `sk_buff` describes every layer

`sk_buff` represents the packet at *all* layers — it's `data` at the app layer,
a `segment` at TCP, a `packet` at IP, and a `frame` at the link layer. If each
layer used its own struct, moving between layers would require copying the payload
over and over, wasting CPU. Instead the kernel uses a single `sk_buff` and just
moves the `data` pointer:

- **On receive**, starting from the driver and going up, the kernel *increases*
  `skb->data` to peel off each protocol header.
- **On send**, it reserves headroom up front and *decreases* `skb->data` at each
  lower layer to prepend that layer's header.

### How many memory copies happen on send?

1. **First** – the send syscall copies the user data into the kernel `sk_buff`.
2. **Second** – with TCP, moving from transport to network layer clones the
   `sk_buff`; the clone goes down and is freed after transmit, while the original
   stays in the transport layer for reliable delivery until the ACK.
3. **Third** – only if the IP layer finds the `sk_buff` larger than the MTU: it
   allocates extra `sk_buff`s and copies the original into several smaller ones.

## Summary

Because networked devices are so heterogeneous, the ISO defined the 7-layer OSI
model — but its complexity meant the simpler 4-layer **TCP/IP model** won out in
practice, and the Linux stack follows it: **application, transport, network, and
link** layers.

- **Sending**: the app hands data to the socket; the stack processes it top-down,
  wrapping it layer by layer, until the NIC puts the frame on the wire.
- **Receiving**: the NIC DMAs the frame in and raises an interrupt; NAPI + softirq
  (`ksoftirqd`) drive the stack bottom-up until the payload lands in the app.

The `sk_buff`, the Ring Buffer, the hard/soft interrupt split, and the NAPI
poll-after-interrupt design are the pieces that make this both correct and fast.
