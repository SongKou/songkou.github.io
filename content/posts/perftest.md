+++
title = 'Perftest_RDMA_Benchmarking'
date = 2026-07-18T20:00:00+08:00
draft = false
categories = ['Network']
tags = ['RDMA', 'RoCE', 'Perftest', 'Linux']
+++

## What is perftest?

**perftest** is the standard micro-benchmark suite for RDMA — it validates and benchmarks InfiniBand, hardware RoCE, and Soft-RoCE links, not ordinary TCP networking. Each tool measures one RDMA verb:

| Test | Measures | Typical use |
|---|---|---|
| `ib_write_bw` | RDMA Write bandwidth | Maximum one-way throughput |
| `ib_write_lat` | RDMA Write latency | Small one-sided write latency |
| `ib_read_bw` | RDMA Read bandwidth | Remote-memory read performance |
| `ib_read_lat` | RDMA Read latency | Storage/database-style remote reads |
| `ib_send_bw` | Send/Receive bandwidth | Message-passing and MPI-style traffic |
| `ib_send_lat` | Send/Receive latency | Small request/response messages |
| `ib_atomic_bw` | Atomic operation rate | Distributed locks and counters |
| `ib_atomic_lat` | Atomic latency | Synchronization performance |

Common use cases:

- Confirming that two RDMA machines can communicate at all
- Comparing InfiniBand, hardware RoCE, and Soft-RoCE
- Measuring latency and bandwidth across different message sizes
- Testing NIC, MTU, queue-depth, and connection settings
- Diagnosing performance regressions after driver or firmware updates
- Burn-in and stability testing
- Measuring GPUDirect RDMA when built with CUDA support

Every test is client–server: run the command with no host argument on one machine (it becomes the server and waits), then run the same command plus the server's IP on the other. The two sides negotiate parameters over ordinary TCP (default port 18515) before the RDMA traffic starts.

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

Reading the header before the numbers:

- **Connection type RC** — Reliable Connection, the default queue-pair type.
- **Link type Ethernet / GID index 1** — this is RoCE; the GID is an IPv4-mapped address (`::ffff:10.10.80.91`), i.e. RoCEv2 over IPv4.
- **Mtu 1024** — the rxe default path MTU, one reason throughput is modest.
- **LID 0000** — LIDs only exist on InfiniBand fabrics; on RoCE they are always zero.
- **Data ex. method: Ethernet** — the out-of-band parameter exchange ran over the normal TCP/IP network.

Result: **58.74 MB/s average** (~0.47 Gb/s) for 64 KiB writes.

## Latency test

Server on ubuntu1, client on ubuntu2, 64-byte messages:

```text
ekou@ubuntu2:~$ ib_send_lat -d rxe0 -s 64 10.10.80.91
---------------------------------------------------------------------------------------
 #bytes #iterations  t_min[usec]  t_max[usec]  t_typical[usec]  t_avg[usec]  t_stdev[usec]  99% percentile[usec]  99.9% percentile[usec]
 64      1000        55.59        1628.06      86.19            111.57       89.26          428.05                1628.06
---------------------------------------------------------------------------------------
```

How to read the columns: `t_typical` (the median-ish value, **86 µs** here) is the honest headline number; `t_avg` is dragged up by outliers; and the gap between `t_typical` and `t_max`/`p99.9` (86 µs → **1.6 ms**!) is the *jitter* — in this lab it is enormous, which is exactly what VM scheduling does to latency.

## Why the VM numbers look like this

These results are perfectly healthy **for this environment** — and would be alarming anywhere else:

| | This lab (Soft-RoCE in VMs) | Why |
|---|---|---|
| Bandwidth | ~59 MB/s | rxe does segmentation, ICRC, and UDP encapsulation in kernel software; the virtual NIC has no RDMA offload |
| Latency | ~86 µs typical | every "RDMA" operation is really a kernel round trip plus a hypervisor network hop |
| Jitter | p99.9 = 1.6 ms | VM vCPU scheduling pauses the guest at will |

Soft-RoCE in a VM validates **functionality** — the stack, the tools, the client–server flow — not **performance**. Which brings us to real hardware.

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
| `--out_json` | machine-readable results |
| `--run_infinitely` | burn-in mode |

Two operational notes: both ends should run the **same perftest version** (the tools exchange versions at startup), and the firewall must allow **TCP 18515** (parameter negotiation) plus **UDP 4791** (the RoCEv2 data itself).

## Summary

- perftest is the go-to for answering "does RDMA work, and how fast?" — one tool per verb, always client–server.
- **Soft-RoCE (`rdma_rxe`) turns any two Linux boxes into an RDMA lab**: `modprobe rdma_rxe` + `rdma link add rxe0 type rxe netdev <if>` and the whole toolchain works — my two VMware VMs moved real RDMA writes at 59 MB/s with 86 µs latency.
- Those VM numbers are functional validation only. On bare-metal ConnectX, expect **near line rate** and **1–2 µs** — provided the GID index, CPU governor, NUMA placement, PCIe slot, and MTU are all right.
- Past back-to-back, performance becomes a fabric property: DSCP/PFC/ECN on the switches decide whether those numbers survive congestion.
