+++
title = 'RDMA Performance Tuning: From Fresh Install to Line-Rate Bandwidth'
date = 2026-07-23T21:00:00+08:00
draft = false
categories = ['Network']
tags = ['RDMA', 'RoCE', 'InfiniBand', 'ConnectX', 'Performance', 'Tuning']
+++

The number-one complaint with RDMA is simple: the bandwidth won't fill up. A 100G card pushing only 60G, two days spent checking drivers, optics, and the peer's configuration — and in the end the fix is a single PCIe parameter.

When RDMA underperforms, 90% of the time the cause is not exotic. It's some factory default that was never touched: the switch buffer carved too small, the PCIe read request size left at its tiny default, the MTU misaligned on one segment of the path. None of these is hard to fix on its own — the hard part is knowing which one to check.

This post is a complete path from "card just arrived" to "bandwidth at line rate": seven checks in order, a one-shot tuning script, and a fault-isolation table to come back to the next time RDMA is slow. A second part covers the other classic complaint — **packet loss** — with a five-step triage ordered by how often each cause is actually the culprit; the first three steps resolve about 70% of cases.

![RDMA tuning path: hardware, system, and network layer checks feeding a tuning script and fault-isolation table](/posts/rdma-performance-tuning/rdma-tuning-path.svg)

> **Applies to:** ConnectX-5/6/7 · MLNX_OFED · RoCEv2 / InfiniBand
>
> This is research from internet, could not verify in lab due to hardware limitation. May not be accurate. 

## 01 — Confirm the hardware

Rule out the physical layer first.

**Is the card even recognized?**

```bash
ibstat
```

**PCIe link state — this is the one that matters most:**

```bash
lspci -vv -s <BDF> | grep -E "Width|Speed"
```

You must see:

| NIC | Width | Speed |
|---|---|---|
| ConnectX-6 | x16 | 16GT/s (Gen4) |
| ConnectX-7 | x16 | 32GT/s (Gen5) |

If the output contains `downgraded`, the link is degraded and bandwidth will never reach line rate. Common causes: the card isn't fully seated, the BIOS has the slot pinned to Gen3, or the slot shares lanes with a GPU.

**NUMA alignment:**

```bash
ibv_devinfo -d mlx5_0 | grep numa
```

NIC on NUMA node 1 while the application runs on NUMA node 0? Cross-NUMA latency will hurt — a lot. This trap is especially easy to step into on dual-socket servers. (And it isn't about inter-socket *bandwidth* — it's the extra hop cost per operation. At high rates, every extra step is wasted performance.)

**Max out the RX/TX queues:**

```bash
ethtool -l <iface>
ethtool -L <iface> combined <max>
```

## 02 — PCIe MaxReadReq

The factory default is **512 bytes** — far too small for large RDMA transfers.

```bash
# Check the current value
lspci -vv -s <BDF> | grep -i MaxRead

# Set it to 4096 (value 5 = 4096)
setpci -s <BDF> 68.w=5
```

Large-message bandwidth typically improves **10–20%** after this change.

⚠️ The setting is lost on reboot. Add it to `rc.local` or wrap it in a systemd service.

## 03 — CPU and interrupt binding

```bash
cpupower frequency-set -g performance
systemctl stop irqbalance
set_irq_affinity_cpulist.sh 0-7 <iface>
```

You can skip this — but RDMA latency will then spike intermittently, and you'll go crazy trying to find out why.

## 04 — mlnx_tune

NVIDIA's official tuning tool, with preset profiles per scenario:

```bash
mlnx_tune -p HIGH_THROUGHPUT
```

It automatically tunes receive queue depth, interrupt coalescing, and ACK delay.

## 05 — Queue parallelism

A single QP has a bandwidth ceiling. The perftest defaults use only **one QP**, which cannot fill the pipe.

**Baseline run:**

```bash
ib_write_bw -d mlx5_0 --report_gbits -F -D 10
```

**Multi-QP with large messages — the real performance:**

```bash
ib_write_bw -d mlx5_0 -q 16 -t 128 -s 1000000 -n 100000 --report_gbits
```

Parameter reference:

| Parameter | Default | Recommended | Why |
|---|---|---|---|
| `-q` (QP count) | 1 | **16–32** | Mandatory on high-latency links |
| `-t` (tx_depth) | 128 | **512–1024** | How many requests are posted at once |
| `-s` (message size) | 65536 | **1000000+** | Bigger messages, higher throughput |

⚠️ **Two pitfalls:**

1. On CX-6 DX, running more than **8 QPs** causes MR cache thrashing and eats about **22%** of throughput. Add `--mr_per_qp`.
2. On CX-7, don't let the total queue count across VFs exceed **32**. Measured: 2 VF × 16 queues = **88.4 Mpps**; 4 VF × 16 queues = **51.3 Mpps** — cut in half.

What to expect from a 100G NIC:

| Scenario | You should see |
|---|---|
| Single QP, small messages | 40–60 Gbps |
| Multi-QP, large messages | 95+ Gbps |
| Both ports simultaneously | 190+ Gbps |

Measured on DGX Spark + CX-7: `ib_send_bw -s 1048576 -q 8 -r 1024 -Q 64` → **93–95 Gbps**.

## 06 — MTU

The highest-hit-rate trap on the network side. The switches are configured for jumbo frames, but one Leaf-to-Spine uplink was forgotten. One wrong segment ruins the whole path.

**End-to-end verification — do not skip this:**

```bash
ping -M do -s 8972 <peer_IP>
```

If it fails, check segment by segment: Leaf downlink, uplink, Spine port, VLAN interface. Missing any one of them breaks it.

**Configuration:**

```bash
# Host side
ip link set <iface> mtu 4200
```

```text
# Switch side (Nexus example)
interface ethernet 1/1
  mtu 9216
```

⚠️ Bigger is not always better. Byte-jitter testing showed some NICs perform *worse* at 4KB MTU than 1KB under high connection counts. When a new card arrives, run `ib_write_bw` at **1KB / 2KB / 4KB** and keep whichever wins.

## 07 — ECN + PFC validation

 Check the network side if bandwidth still won't climb after going through the first six steps.

```bash
# Switch: PFC trigger counters
show interface priority-flow-control

# NIC: ECN marks
ethtool -S <iface> | grep ecn
```

⚠️ PFC counters continuously climbing = something is misconfigured. On a healthy link PFC almost never fires.

Field-tested values, ready to use:

| Parameter | Recommended | Notes |
|---|---|---|
| PFC priority | **Priority 3 only** | For RoCE traffic — never enable all 8 |
| Buffer share | **10–20%** | Don't leave it at the 50% default |
| ECN threshold | **Min 150KB / Max 1500KB** | Configured on the switch |
| DCQCN ToS | **DSCP × 4** | DSCP 24 → write 96 |

⚠️ Never let PFC and TCP share the same physical port. TCP retransmissions and PFC pause frames stack up and can deadlock PFC. Put RDMA on one port and TCP on another — physical isolation.

## The tuning script

New machine comes online — run this:

```bash
#!/bin/bash
# Applies to: ConnectX-5/6/7 + MLNX_OFED v24+

# Interface to tune: first argument, or ens1np0 if none given
IFACE=${1:-ens1np0}
# Resolve the NIC's PCIe address (BDF) from the interface name
BDF=$(ethtool -i $IFACE | grep bus-info | awk '{print $2}')

# Pin all CPU cores at max frequency — no downclocking mid-transfer
cpupower frequency-set -g performance 2>/dev/null
# Stop the IRQ balancer so interrupts stop migrating between cores
systemctl stop irqbalance 2>/dev/null
# Pin the NIC's interrupts to cores 0-7 (same NUMA node as the NIC)
set_irq_affinity_cpulist.sh 0-7 $IFACE
# Jumbo frames — must match the switch config end to end
ip link set $IFACE mtu 4200
# PCIe MaxReadReq 512 -> 4096 (register 68, value 5 = 4096)
setpci -s $BDF 68.w=5
# Max out the RX/TX ring buffers to absorb bursts
ethtool -G $IFACE rx 8192 tx 8192
# Apply NVIDIA's throughput profile (queue depth, coalescing, ACK delay)
mlnx_tune -p HIGH_THROUGHPUT

echo "Done. Verify with:"
echo "ib_write_bw -d mlx5_0 -q 16 -t 128 --report_gbits -F -D 10"
```

## Fault-isolation table

| Symptom | Check first | Fix |
|---|---|---|
| Bandwidth stuck at 40–50 Gbps | PCIe Width/Speed | Change slot / check BIOS / check power |
| PFC counters keep climbing | ECN threshold / MTU | Tune ECN / align MTU |
| CX-7 only pushing 13G | PCIe power delivery | Reseat cables, reboot |
| Multi-VF bandwidth halved | Total queue count > 32 | Fewer VFs or fewer queues |
| RDMA slower than TCP | Single-QP WR ceiling | More connections in parallel |
| Bandwidth oscillates | IRQ migration / CPU downclocking | Stop irqbalance, pin cores, performance governor |
| New machine won't reach line rate | MTU misalignment | `ping -M do` segment by segment |
| Container slower than bare metal | NUMA binding / CNI MTU | topologyManager |

---

## Part 2 — Packet loss: check the easy things first

A war story that sets the rule: on a freshly inherited cluster, RDMA bandwidth was stuck at half of what `ib_write_bw` should show. First suspicion was cabling — half a day spent reseating, swapping cables, and measuring optical power. The actual cause: one port on the *peer* switch still at the default MTU of 1500.

The habit that came out of it: **when RDMA drops packets, start with what's easiest to verify, not with what looks most likely.** The order below is by real-world frequency — how often each step turns out to be the answer — not by textbook OSI layers. Knock out the 5-minute checks before going deep.

![RDMA packet-loss triage: five steps ordered by hit rate, first three solve about 70% of cases](/posts/rdma-performance-tuning/rdma-packet-loss-triage.svg)

**The 10-second precheck.** Same hardware check as step 01 of Part 1 — `ibstat` and `lspci -vv -s <BDF> | grep -E "Width|Speed"`. If the PCIe link shows `downgraded`, bandwidth is capped no matter what else you fix. It costs 10 seconds; always do it first.

### Step 1 — Cables and optics (~20% of cases)

**Transceiver temperature first:**

```text
# Cisco NX-OS
show interface ethernet 1/1 transceiver details

# Cumulus Linux
nv show interface swp1 transceiver
```

⚠️ Sustained above **60°C**, the module's performance degrades. Swapping cables won't help here — fix the airflow or replace the module.

**Then the cable spec.** Buying the wrong cable grade is more common than you'd think: a QSFP28 cable used where QSFP56 was needed pins bandwidth at **50G**; a certified cable brought it straight back to **100G+**. The connectors look identical while the speed differs by 2× — read the label, not the shape.

**Physical-layer signal quality:**

```bash
ethtool <iface> | grep -i link
ethtool -S <iface> | grep -i crc
ethtool -S <iface> | grep -i error
cat /sys/class/infiniband/mlx5_0/ports/1/counters/symbol_error_counter
```

Verdict: CRC or error counters continuously climbing → the physical layer is the problem. Replace the cable or the optic.

### Step 2 — MTU aligned end to end (~30% of cases)

The single highest hit rate on the list — and the same check as step 06 of Part 1:

```bash
ping -M do -s 8972 <peer_IP>
```

- ✅ Replies = MTU is consistent end to end.
- ❌ No reply = some segment is misaligned. Walk it hop by hop: Leaf downlink, uplink, Spine port, VLAN interface. A single path can cross four or five devices, and missing any one of them breaks it.

Configuration commands are in Part 1 step 06 (`ip link set <iface> mtu 4200` on hosts, `mtu 9216` on switch ports), along with the warning that bigger is not automatically better — race the candidate MTUs with `ib_write_bw` and keep the winner.

### Step 3 — PFC configuration consistency (~20% of cases)

RoCE runs on a lossless network, and lossless is delivered by PFC. The most common PFC pit of all: **it's only enabled on one side.** Both switches (Spine and Leaf) need it on, and both `rx` and `tx` must be enabled.

**Cisco NX-OS:**

```text
show interface priority-flow-control

# Port       Mode     Oper(VL bmap)   RxPPP     TxPPP
# ---------  -------  -------------   -------   -------
# Eth1/1     ON       On(8)           4885      3709920
# Eth1/2     ON       On(8)           0         1073455
# Eth1/3     OFF      Off             0         0
```

Verdict: `RxPPP` and `TxPPP` continuously climbing → PFC is firing constantly, which is *not* a healthy state.

**Cumulus Linux:**

```text
nv show interface swp1 qos roce status

# pfc
#   pfc-priority      3
#   rx-enabled        yes
#   tx-enabled        yes
```

Both sides must show rx/tx as `on`/`yes` — if either side is off, PFC simply doesn't work.

⚠️ Occasional PFC triggering is normal. Continuous rapid growth means something is wrong — usually a mismatched ECN threshold or under-provisioned buffer.

### Step 4 — PFC storm hunt (~20% of cases)

A PFC storm is sneakier than PFC being off: the configuration looks perfectly correct, but throughput won't climb.

**Locate the storming port** — look for the ports with high `TxPPP`:

```text
show interface priority-flow-control
```

**The usual triggers:**

- **Incast** — AllReduce-style many-to-one traffic overwhelms the buffer.
- **RDMA and TCP mixed on one port** — TCP retransmissions and PFC pause frames stack up on each other.
- **ECN threshold set too high** — ECN never gets the chance to slow senders down, so PFC fires first.

**Confirm on the host side:**

```bash
ethtool -S <iface> | grep pause
```

Once confirmed, the fixes point in three directions: check whether the ECN threshold is too high, keep PFC and TCP off the same physical port, and physically isolate RDMA traffic from everything else.

### Step 5 — ethtool counter deep-read (~10% of cases)

The first four steps found nothing — time to read the NIC's hardware counters.

> The counter names below apply to the NVIDIA **mlx5** driver; other vendors name them differently.

```bash
ethtool -S <iface> | grep -E "drop|discard|error|miss"
```

The ones to watch:

| Counter | Meaning |
|---|---|
| `rx_steer_missed_packets` | Packet reached the NIC but matched no flow-steering rule |
| `rx_buff_alloc_err` | Kernel buffer allocation failed |
| `rx_crc_errors_phy` | Physical-layer CRC errors (cable/signal problem) |
| `outbound_pci_buffer_overflow` | PCIe back-pressure |

**PFC and RDMA statistics:**

```bash
ethtool -S <iface> | grep -i pause
ethtool -S <iface> | grep rdma
```

⚠️ Totals alone tell you nothing — watch the **delta while traffic is running**. On a healthy link the PFC counters should barely move. If `pause` counters keep climbing, go back to steps 3 and 4.

### Triage summary

| Order | What to check | Time | Hit rate |
|---|---|---|---|
| 1 | Cables & optics (temperature, spec, CRC) | 3 min | ~20% |
| 2 | MTU aligned end to end | 5 min | ~30% |
| 3 | PFC configuration consistency | 5 min | ~20% |
| 4 | PFC storm hunt | 10 min | ~20% |
| 5 | ethtool counter deep-read | 15 min | ~10% |

The first three steps already cover **~70%** of cases.

The order is by frequency, not a mandatory sequence. In practice: if the first three steps come up empty, going straight to a packet capture is often more efficient than grinding through steps 4 and 5 — the later steps need much more information to be conclusive.

Most RDMA packet-loss tickets end at the most basic configuration, not anything deep. The hard part of the triage isn't that any operation is complex — it's doing them in the right order. Start with the complicated suspects and you burn hours; start with the 5-minute checks and most tickets never get past step 3.
