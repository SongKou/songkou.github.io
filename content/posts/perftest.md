+++
title = 'Perftest_RDMA_Benchmarking'
date = 2026-07-18T20:00:00+08:00
draft = false
categories = ['Network']
tags = ['RDMA', 'RoCE', 'Perftest', 'Linux']
+++

## What is perftest?

**perftest** is the standard micro-benchmark suite for RDMA — it validates and benchmarks InfiniBand, hardware RoCE, and Soft-RoCE links, not ordinary TCP networking. Each tool measures one RDMA verb:

| Test | Measures | Real-world workloads it models |
|---|---|---|
| `ib_write_bw` | RDMA Write bandwidth | Data replication, storage writes, GPU/bulk data transfers |
| `ib_write_lat` | RDMA Write latency | Updating remote memory — small one-sided writes |
| `ib_read_bw` | RDMA Read bandwidth | Distributed storage and database reads, remote memory access |
| `ib_read_lat` | RDMA Read latency | Fetching remote memory — storage/database-style lookups |
| `ib_send_bw` | Send/Receive bandwidth | MPI messages, RPC, request queues |
| `ib_send_lat` | Send/Receive latency | Small request/response round trips — RPC and MPI latency |
| `ib_atomic_bw` | Atomic operation rate | Distributed locks and counters |
| `ib_atomic_lat` | Atomic latency | Synchronization primitives |

### Write, Read, and Send are different operations

The grouping in the table is not accidental — the three verbs have different semantics:

**RDMA Write** — the client writes directly into registered memory on the server. The server CPU does not process each operation:

```text
Client memory --write--> Server memory
```

Commonly used for high-throughput replication and pushing data into remote buffers.

**RDMA Read** — the client fetches data directly from registered server memory, again without the remote CPU in the data path:

```text
Client memory <--read-- Server memory
```

Useful for distributed caches, databases, remote-memory systems, and storage reads.

**Send/Receive** — a **two-sided** operation: the client sends a message and the server must already have a receive buffer posted:

```text
Client Send --message--> Server Receive
```

This is closer to RPC or MPI messaging — which is why the one-sided verbs model memory/storage access while the send tests model message exchange.

Common use cases:

- Confirming that two RDMA machines can communicate at all
- Comparing InfiniBand, hardware RoCE, and Soft-RoCE
- Measuring latency and bandwidth across different message sizes
- Testing NIC, MTU, queue-depth, and connection settings
- Diagnosing performance regressions after driver or firmware updates
- Burn-in and stability testing
- Measuring GPUDirect RDMA when built with CUDA support

Every test is client–server, always following the same pattern:

```bash
# Server: start and wait
ib_write_bw [options]

# Client: same command, same options, plus the server's IP
ib_write_bw [options] SERVER_IP
```

Use the same test and the same options on both machines. The two sides negotiate parameters over ordinary TCP (default port 18515) before the RDMA traffic starts.

## The lab: two Ubuntu VMs with Soft-RoCE

My test bed is deliberately modest — two Ubuntu Server 24.04 VMs on VMware Workstation 16. There is no RDMA hardware here, which is exactly the point: **Soft-RoCE (the `rdma_rxe` kernel driver)** implements RoCEv2 entirely in software on top of a plain Ethernet NIC, so you can learn and validate the whole RDMA toolchain on any machine.

```text
VMware Workstation 16 (one host)

+---------------------------+     +---------------------------+
| ubuntu1 (Ubuntu 24.04)    |     | ubuntu2 (Ubuntu 24.04)    |
| ens33  10.10.80.91        |     | ens33  10.10.80.92        |
| rxe0 (Soft-RoCE on ens33) |     | rxe0 (Soft-RoCE on ens33) |
+------------+--------------+     +------------+--------------+
             |                                 |
             +---------- VMware vmnet ---------+
```

### Install the tooling

On **both** VMs:

```text
ekou@ubuntu1:~$ sudo apt install perftest
The following NEW packages will be installed:
  libibumad3 librdmacm1t64 perftest
Setting up perftest (24.01.0+0.38-1build2) ...

ekou@ubuntu1:~$ ib_write_bw --version
Version: 6.20

ekou@ubuntu1:~$ sudo apt install rdma-core ibverbs-utils
Setting up ibverbs-utils (50.0-2ubuntu0.2) ...
Setting up rdma-core (50.0-2ubuntu0.2) ...
```

`perftest` brings the benchmarks; `rdma-core` + `ibverbs-utils` bring the RDMA stack and the inspection tools (`ibv_devices`, `ibv_devinfo`, `rdma`).

### Create the Soft-RoCE device

A fresh VM has no RDMA devices — `ibv_devices` prints an empty table and `ibv_devinfo` says `No IB devices found`. Load the `rdma_rxe` driver and attach an rxe device to the Ethernet interface (again on **both** VMs):

```text
ekou@ubuntu1:~$ ip -br link
lo               UNKNOWN        00:00:00:00:00:00 <LOOPBACK,UP,LOWER_UP>
ens33            UP             00:0c:29:a0:4a:ce <BROADCAST,MULTICAST,UP,LOWER_UP>

ekou@ubuntu1:~$ sudo modprobe rdma_rxe
ekou@ubuntu1:~$ sudo rdma link add rxe0 type rxe netdev ens33

ekou@ubuntu1:~$ ibv_devices
    device                 node GUID
    ------              ----------------
    rxe0                020c29fffea04ace
```

The node GUID is derived from the NIC's MAC address (`00:0c:29:a0:4a:ce` → `020c29fffea04ace` — the flipped bit plus `fffe` in the middle is the standard EUI-64 expansion). Note that `rdma link add` does not survive a reboot; re-add it (or automate it with a systemd unit) after restarting.

## Bandwidth test

Server side (ubuntu1) — start and wait:

```text
ekou@ubuntu1:~$ ib_write_bw -d rxe0

************************************
* Waiting for client to connect... *
************************************
```

Client side (ubuntu2) — same command plus the server IP:

```text
ekou@ubuntu2:~$ ib_write_bw -d rxe0 10.10.80.91
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : rxe0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 1024[B]
 Link type       : Ethernet
 GID index       : 1
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0000 QPN 0x0011 PSN 0x798a38 RKey 0x000249 VAddr 0x00766af95ef000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:10:80:92
 remote address: LID 0000 QPN 0x0011 PSN 0xf4a060 RKey 0x00020b VAddr 0x0075769ae7d000
 GID: 00:00:00:00:00:00:00:00:00:00:255:255:10:10:80:91
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 65536      5000             66.81              58.74              0.000940
---------------------------------------------------------------------------------------
```

That output has three parts: a header describing the test setup, two address lines describing the connection, and the result row.

### Reading the header

| Field | Meaning |
| --- | --- |
| `Dual-port : OFF` | Testing one port only; `-O` runs both ports of a dual-port adapter |
| `Device : rxe0` | The RDMA device under test — not the Ethernet interface name |
| `Number of qps : 1` | A single queue pair. On fast links one QP is often the ceiling; `-q` adds more |
| `Transport type : IB` | The InfiniBand *transport*, even though the link is Ethernet — see below |
| `Connection type : RC` | Reliable Connection: ordered and acknowledged, the default queue-pair type |
| `Using SRQ : OFF` | No shared receive queue |
| `TX depth : 128` | Send-queue depth — how many work requests may be outstanding at once |
| `CQ Moderation : 1` | Completions reported one at a time; the `-R` runs below show `100`, batching them to cut CPU overhead |
| `Mtu : 1024[B]` | The rxe default path MTU, one reason throughput here is modest |
| `Link type : Ethernet` | This is RoCE, not InfiniBand |
| `GID index : 1` | Which GID table entry identifies this endpoint |
| `rdma_cm QPs : OFF` | Connected directly rather than through RDMA-CM (`-R` switches this to `ON`) |
| `Data ex. method : Ethernet` | The out-of-band parameter exchange ran over ordinary TCP/IP |

**`Transport type : IB` on an Ethernet link is not a contradiction** — it is the whole idea of RoCE. The InfiniBand transport (queue pairs, RC semantics, the BTH header) rides inside UDP/IP over Ethernet, so perftest reports the transport and the link layer separately and they legitimately disagree. My [RoCE on Cumulus Linux](/posts/roce_cumulus_linux/) post draws that encapsulation stack.

### Reading the address lines

| Field | Meaning |
| --- | --- |
| `LID 0000` | Local Identifier, an InfiniBand fabric address — always zero on RoCE |
| `QPN 0x0011` | Queue-Pair Number, the endpoint's identifier within the device |
| `PSN 0x798a38` | Initial Packet Sequence Number, randomized per connection |
| `RKey 0x000249` | Remote key — the token authorizing the peer to access the registered buffer |
| `VAddr 0x00766af95ef000` | Virtual address of that buffer in the local process |
| `GID: ...255:255:10:10:80:92` | The endpoint's global identifier |

`RKey` and `VAddr` are what make an RDMA Write **one-sided**: the peer is handed a memory address plus a key and writes straight into that memory, with no involvement from the remote CPU. That is the mechanism behind the whole "bypass the CPU" claim.

The GID is printed as 16 bytes in decimal. Read the tail — `...:255:255:10:10:80:92` is `::ffff:10.10.80.92`, an IPv4-mapped IPv6 address, which is what marks this as **RoCEv2 over IPv4**.

### Reading the result row

| Column | Meaning |
| --- | --- |
| `#bytes 65536` | Message size — 64 KiB, the default for the bandwidth tests |
| `#iterations 5000` | How many messages were sent (`-n`) |
| `BW peak 66.81` | Best instantaneous rate observed |
| `BW average 58.74` | Sustained rate across the whole run — the number to quote |
| `MsgRate 0.000940` | Millions of messages per second |

The three numbers are internally consistent, and checking that is a good habit: `0.000940 Mpps` is 940 messages/s, and `940 x 65536 B = 61.6 MB/s = 58.75 MiB/s` — matching the reported 58.74. That also settles what perftest means by "MB/sec": it is **MiB/sec (2^20 bytes)**, not 10^6, which is why converting to line rate needs the factor from the `--report_gbits` note. So this run is **58.74 MiB/s ≈ 0.49 Gb/s** for 64 KiB writes.

A wide gap between peak and average means an unstable run. Here 66.81 vs 58.74 is the VM scheduler intruding — on quiet hardware the two converge.

## Latency test

Server on ubuntu1, client on ubuntu2, 64-byte messages:

```text
ekou@ubuntu2:~$ ib_send_lat -d rxe0 -s 64 10.10.80.91
---------------------------------------------------------------------------------------
 #bytes #iterations  t_min[usec]  t_max[usec]  t_typical[usec]  t_avg[usec]  t_stdev[usec]  99% percentile[usec]  99.9% percentile[usec]
 64      1000        55.59        1628.06      86.19            111.57       89.26          428.05                1628.06
---------------------------------------------------------------------------------------
```

Nine columns, and they tell different stories:

| Column | This run | Meaning |
| --- | --- | --- |
| `#bytes` | 64 | Message size (`-s`) — latency tests use small messages |
| `#iterations` | 1000 | Number of round trips measured (`-n`) |
| `t_min` | 55.59 µs | Best case — the floor when nothing interferes |
| `t_max` | 1628.06 µs | Worst single sample observed |
| `t_typical` | 86.19 µs | The most representative value — the honest headline number |
| `t_avg` | 111.57 µs | Arithmetic mean, dragged upward by outliers |
| `t_stdev` | 89.26 µs | Spread of the samples |
| `99% percentile` | 428.05 µs | 99 of 100 round trips finished within this |
| `99.9% percentile` | 1628.06 µs | Tail latency — what the unluckiest 1-in-1000 request saw |

Quote **`t_typical`**, not `t_avg`: here the mean (111 µs) is nearly 30% higher than the typical value (86 µs) purely because of outliers. And a `t_stdev` almost as large as `t_avg` is itself the warning — on a stable path the standard deviation is a small fraction of the mean.

The real story is in the tail. From 86 µs typical to **1.6 ms** at p99.9 is roughly 19× — and note `t_max` equals `99.9% percentile` exactly, meaning the single worst sample *is* the tail. That spread is jitter, and in this lab it is the VM scheduler pausing the guest mid-measurement. On tuned bare metal you would expect the percentiles to sit within a small multiple of `t_typical`.

One measurement detail: latency tools time a full round trip and report **half** of it as the estimated one-way latency.

## More recipes: duration, iterations, and all sizes

My runs above used the defaults (5000 iterations of one 64 KiB size, direct GID exchange). Three variations worth knowing — shown here with RDMA-CM connection setup (`-R`), which is also what enables ToS/DSCP marking with `-T`. You can spot the difference in the output header: with `-R` the test reports `rdma_cm QPs : ON` and `Data ex. method : rdma_cm` instead of the `OFF` / `Ethernet` of the direct runs.

**Bandwidth for a fixed duration** — 64 KiB messages for 10 seconds:

```bash
# Server:
ib_write_bw -d rxe0 -R -s 65536 -D 10
# Client:
ib_write_bw -d rxe0 -R -s 65536 -D 10 10.10.80.91
```

`ib_read_bw` and `ib_send_bw` follow exactly the same pattern — swap the command name to benchmark the Read or Send verb instead.

**Latency with a larger sample** — 10,000 exchanges of 64-byte messages:

```bash
# Server:
ib_write_lat -d rxe0 -R -s 64 -n 10000
# Client:
ib_write_lat -d rxe0 -R -s 64 -n 10000 10.10.80.91
```

Again, `ib_read_lat` and `ib_send_lat` are identical in usage. Note how the latency tools measure: they perform a round trip and report **half of it** as the estimated one-way latency.

The `-D` duration flag works on latency tests too. A real 60-second run from the lab:

```text
ekou@ubuntu2:~$ ib_send_lat -d rxe0 -s 64 -D 60 10.10.80.91
---------------------------------------------------------------------------------------
                    Send Latency Test
 Connection type : RC           Using SRQ      : OFF
 Mtu             : 1024[B]
 Link type       : Ethernet
---------------------------------------------------------------------------------------
 #bytes        #iterations       t_avg[usec]    tps average
 64            151456            99.03          5048.78
---------------------------------------------------------------------------------------
```

Duration mode reports only the average and the transactions-per-second — no percentile columns. The numbers are self-consistent: 99.03 µs one-way means a ~198 µs round trip, and 1 / 198 µs ≈ the 5,049 tps reported. Over 151,456 exchanges the 99 µs average also agrees with the earlier short run (t_typical 86 µs plus the long jitter tail pulling the mean up).

**Sweep every message size** — `-a` runs from 2 B up to 8 MiB, revealing where the link saturates:

```bash
# Server:
ib_write_bw -d rxe0 -R -a
# Client:
ib_write_bw -d rxe0 -R -a 10.10.80.91
```

That sweep on the lab produced:

```text
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 2          5000             0.09               0.06               0.028957
 4          5000             0.23               0.11               0.029573
 8          5000             0.49               0.26               0.033853
 16         5000             0.89               0.43               0.028452
 32         5000             1.92               1.11               0.036419
 64         5000             4.00               2.22               0.036406
 128        5000             7.09               4.08               0.033438
 256        5000             16.17              8.19               0.033541
 512        5000             26.46              14.07              0.028824
 1024       5000             59.16              31.37              0.032124
 2048       5000             63.48              36.53              0.018704
 4096       5000             72.21              36.67              0.009388
 8192       5000             77.82              42.38              0.005424
 16384      5000             46.29              35.53              0.002274
 32768      5000             51.99              40.16              0.001285
 65536      5000             43.33              39.65              0.000634
 131072     5000             45.07              39.31              0.000314
 262144     5000             40.65              36.55              0.000146
 524288     5000             37.57              37.19              0.000074
 1048576    5000             37.80              37.08              0.000037
 2097152    5000             40.55              38.46              0.000019
 4194304    5000             109.60             75.31              0.000019
 8388608    5000             113.01             107.23             0.000013
---------------------------------------------------------------------------------------
```

Reading the curve:

- **2 B – 1 KiB: operation-rate bound.** The message rate stays flat (~0.03 Mpps ≈ 30k ops/s) while bandwidth grows almost linearly with size — the signature of a per-operation bottleneck. rxe's software path caps how many operations per second it can push, so a bigger message simply carries more bytes per operation.
- **2 KiB – 2 MiB: plateau** around 36–42 MB/s.
- **4–8 MiB: jump to 75–107 MB/s.** Per-operation overhead is now fully amortized and VM scheduling jitter averages out across each long transfer; peak and average converging (113 vs 107) marks a stable run.
- On bare-metal ConnectX the same sweep instead climbs quickly and pins at line rate for every large size — a very different shape from this software curve.

Flag recap for these recipes: `-d` selects the RDMA device, `-R` connects through the RDMA Connection Manager, `-s` sets the message size, `-D` runs for a fixed number of seconds (pairs nicely with `--cpu_util`), and `-n` sets the iteration count.

## Atomic operations: ib_atomic_bw and ib_atomic_lat

`ib_atomic_bw` and `ib_atomic_lat` benchmark **remote RDMA atomic operations**. An atomic modifies an 8-byte value in remote registered memory as one indivisible operation, so no other participant can ever observe a partially updated value. perftest supports two atomic types:

| Atomic type | Behavior | Common use |
|---|---|---|
| `FETCH_AND_ADD` | Returns the old value and adds a number | Shared counters, sequence numbers, work allocation |
| `CMP_AND_SWAP` | Replaces a value only if it equals an expected value | Distributed locks, ownership, synchronization |

### ib_atomic_bw

Measures how many remote atomic operations complete per second. For atomics the **operation/message rate is more meaningful than data bandwidth** — every operation touches only 8 bytes.

```bash
# Server:
ib_atomic_bw -d rxe0 -R -A FETCH_AND_ADD -D 10
# Client:
ib_atomic_bw -d rxe0 -R -A FETCH_AND_ADD -D 10 10.10.80.91
```

On the lab this returned:

```text
ekou@ubuntu2:~$ ib_atomic_bw -d rxe0 -R -A FETCH_AND_ADD -D 10 10.10.80.91
 WARNING: BW peak won't be measured in this run.
Device not recognized to implement inline feature. Disabling it
---------------------------------------------------------------------------------------
                    Atomic FETCH_AND_ADD BW Test
 Connection type : RC           Using SRQ      : OFF
 Outstand reads  : 128
 rdma_cm QPs     : ON
 Data ex. method : rdma_cm
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 8          578000           0.00               0.73               0.096329
---------------------------------------------------------------------------------------
```

This is the op-rate point made concrete: the "bandwidth" is a laughable **0.73 MB/s**, but the message rate is **0.096 Mpps ≈ 96,000 atomic operations per second** (578,000 in 10 s) — for 8-byte synchronization operations, the second number is the one that matters. The two warnings are normal: peak bandwidth is not computed in duration mode, and rxe has no inline-data support.

Throughput often scales with the number of in-flight requests — allow multiple outstanding atomics with `-o`:

```bash
# Server:
ib_atomic_bw -d rxe0 -R -A FETCH_AND_ADD -o 16 -D 10
# Client:
ib_atomic_bw -d rxe0 -R -A FETCH_AND_ADD -o 16 -D 10 10.10.80.91
```

(A confession from my own captures: I accidentally started the server with `-o 16` but the client without it — each side's header showed its own `Outstand reads` value and the test still completed. It ran, but it violates the "same options on both machines" rule; keep them identical so the header tells you what was actually measured.)

### ib_atomic_lat

Measures the time to complete one remote atomic operation:

```bash
# Compare-and-Swap latency (server / client):
ib_atomic_lat -d rxe0 -R -A CMP_AND_SWAP -n 10000
ib_atomic_lat -d rxe0 -R -A CMP_AND_SWAP -n 10000 10.10.80.91

# Fetch-and-Add latency: same pattern
ib_atomic_lat -d rxe0 -R -A FETCH_AND_ADD -n 10000 10.10.80.91
```

The new flags beyond the earlier recipes: `-A` selects the atomic operation type, and `-o` sets the number of outstanding requests.

Two practical notes:

- Atomics require the **reliable-connected (RC)** transport — perftest's default, so nothing to change. Check whether your device exposes atomic support at all:

```bash
ibv_devinfo -v | grep -i atomic
```

On the Soft-RoCE lab:

```text
ekou@ubuntu2:~$ ibv_devinfo -v | grep -i atomic
        atomic_cap:                     ATOMIC_HCA (1)
```

`ATOMIC_HCA` means the device guarantees atomicity between its own operations — rxe supports atomics, which is why the tests above work at all.

- These tests represent **synchronization and metadata operations, not bulk data**. Use `ib_write_bw` or `ib_read_bw` for data throughput.

## Why the VM numbers look like this

These results are perfectly healthy **for this environment** — and would be alarming anywhere else:

| | This lab (Soft-RoCE in VMs) | Why |
|---|---|---|
| Bandwidth | ~59 MB/s | rxe does segmentation, ICRC, and UDP encapsulation in kernel software; the virtual NIC has no RDMA offload |
| Latency | ~86 µs typical | every "RDMA" operation is really a kernel round trip plus a hypervisor network hop |
| Jitter | p99.9 = 1.6 ms | VM vCPU scheduling pauses the guest at will |

Soft-RoCE in a VM validates **functionality** — the stack, the tools, the client–server flow — not **performance**. Which brings us to real hardware.

It also cannot validate a *lossless* configuration. PFC is a hardware pause mechanism between a real NIC and a real switch port, and ECN marking happens in switch buffers — `rxe` over a virtual NIC has neither. So the PFC pause and CNP counters that matter in a production RoCE fabric are simply absent or permanently zero here, and no perftest run in this lab can tell you whether PFC, ECN or DCQCN are configured correctly. That verification needs hardware; see [End-to-End RoCEv2 Configuration](/posts/rocev2-cisco-cumulus-connectx-end-to-end/) for the counters to watch and where to read them.

## On real servers: bare-metal with a ConnectX adapter

On a physical server with an NVIDIA/Mellanox ConnectX NIC, the same commands apply almost unchanged — the differences are the device name, the GID selection, and the numbers you should expect.

### Verify the RDMA device

With a ConnectX adapter, the inbox `rdma-core` stack already exposes the device (install MLNX_OFED / DOCA-Host for the full feature set). No `modprobe rdma_rxe` needed — the hardware device is just there:

```bash
ibv_devices              # expect e.g. mlx5_0
ibv_devinfo              # check: state PORT_ACTIVE, active_mtu 4096, link_layer Ethernet
rdma link show
show_gids                # MLNX_OFED tool: list GIDs with their types
```

Conceptual `show_gids` output:

```text
DEV     PORT  INDEX  GID                       IPv4         VER   NETDEV
mlx5_0  1     0      fe80::...:abcd            —            v1    ens6f0np0
mlx5_0  1     1      fe80::...:abcd            —            v2    ens6f0np0
mlx5_0  1     2      ::ffff:192.168.100.11     192.168...   v1    ens6f0np0
mlx5_0  1     3      ::ffff:192.168.100.11     192.168...   v2    ens6f0np0
```

Pick the **RoCEv2 GID with your IPv4 address** (index 3 above) and pass it with `-x`. A wrong GID index is the classic "ping works, RDMA fails" cause — the test either hangs or dies at connection setup.

### Run the same tests

Server:

```bash
ib_write_bw -d mlx5_0 -a -F --report_gbits
```

Client:

```bash
ib_write_bw -d mlx5_0 -a -F --report_gbits -x 3 192.168.100.11
```

- `-a` sweeps message sizes from 2 B to 8 MiB, exposing the size where the link saturates
- `-F` suppresses the CPU-frequency warning (but do set the governor — see below)
- `--report_gbits` reports Gb/s instead of MiB/s

### What to expect on hardware

Rough, well-tuned back-to-back figures (large messages for bandwidth, small for latency):

| Link | `ib_write_bw` (large msgs) | `ib_write_lat` typical |
|---|---|---|
| 25 GbE ConnectX | ~23 Gb/s | ~1–2 µs |
| 100 GbE ConnectX | ~96–97 Gb/s | ~1–2 µs |
| 200/400 GbE | near line rate (may need `-q` > 1) | ~1–2 µs |

You never see the full line rate as payload — Ethernet/IP/UDP/BTH headers and ICRC consume a few percent. If you are far below these numbers, suspect the tuning list next.

### Bare-metal tuning checklist

- **CPU governor**: `cpupower frequency-set -g performance` — perftest even warns about this (that is what `-F` silences).
- **NUMA locality**: find the NIC's node with `cat /sys/class/infiniband/mlx5_0/device/numa_node`, then pin: `numactl --cpunodebind=<node> --membind=<node> ib_write_bw ...`. Crossing the inter-socket link costs real bandwidth.
- **PCIe**: confirm width/generation with `lspci -vv` (`LnkSta`); a x16 Gen4 slot is ~252 Gb/s raw — a 400G NIC in a x8 slot cannot deliver.
- **MTU**: jumbo frames end to end (host 9000, switch 9216); `ibv_devinfo` should show `active_mtu 4096`.
- **Multiple queue pairs**: `-q 4` (and beyond) helps saturate very fast links and exercises ECMP entropy on routed fabrics.
- **Duration mode for stability**: `-D 30 --cpu_util` gives a sustained-rate view with CPU cost; `--run_infinitely` for burn-in.
- **Hugepages**: `--use_hugepages` reduces memory-translation overhead at large scale.

### Multi-hop fabrics

Back-to-back tests isolate the NICs. Across a switched fabric, results also depend on the switch QoS config — DSCP marking, PFC on the lossless priority, and ECN thresholds. That is a topic of its own: see my [RoCE on Cumulus Linux](/posts/roce_cumulus_linux/) post, which covers the switch side and the host `mlnx_qos`/`cma_roce_tos` marking commands that make `-R -T 106` (RDMA-CM + ToS marking) meaningful.

## The flags worth knowing

The full `-h` output is long; these are the ones I actually reach for (perftest 6.20):

| Flag | Meaning |
|---|---|
| `-d <dev>` / `-i <port>` | choose device / port |
| `-a` | sweep all message sizes 2 B – 8 MiB |
| `-s <size>` / `-n <iters>` | one size / iteration count |
| `-b` | bidirectional — **must be on both server and client** |
| `-q <n>` | number of queue pairs |
| `-c RC\|UC\|XRC\|DC` | connection type (default RC) |
| `-D <sec>` + `--cpu_util` | duration mode with CPU utilization |
| `-R` + `-T <tos>` | RDMA-CM connect + ToS/DSCP marking (`-T` only works with `-R`; lowercase `-t` is tx-depth!) |
| `-x <gid>` | GID index (RoCEv1 vs v2, IPv4 vs IPv6) |
| `-F` | ignore the CPU-frequency warning |
| `--report_gbits` | Gb/s instead of MiB/s |
| `-H` | (latency tests) print the full histogram |
| `-W <counters>` | report the change in NIC/RDMA counters across the run, e.g. `-W counters/port_xmit_data,hw_counters/out_of_buffer` |
| `--out_json` | machine-readable results |
| `--run_infinitely` | burn-in mode |

Two operational notes: both ends should run the **same perftest version** (the tools exchange versions at startup), and the firewall must allow **TCP 18515** (parameter negotiation) plus **UDP 4791** (the RoCEv2 data itself).

## Summary

- perftest is the go-to for answering "does RDMA work, and how fast?" — one tool per verb, always client–server.
- **Soft-RoCE (`rdma_rxe`) turns any two Linux boxes into an RDMA lab**: `modprobe rdma_rxe` + `rdma link add rxe0 type rxe netdev <if>` and the whole toolchain works — my two VMware VMs moved real RDMA writes at 59 MB/s with 86 µs latency.
- Those VM numbers are functional validation only. On bare-metal ConnectX, expect **near line rate** and **1–2 µs** — provided the GID index, CPU governor, NUMA placement, PCIe slot, and MTU are all right.
- Past back-to-back, performance becomes a fabric property: DSCP/PFC/ECN on the switches decide whether those numbers survive congestion.
