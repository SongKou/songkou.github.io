+++
title = 'NCCL and NVLink: Collective Communication and the Two Domains of an AI Cluster'
date = 2026-08-05T21:00:00+08:00
draft = false
categories = ['Network']
tags = ['NCCL', 'NVLink', 'NVSwitch', 'AllReduce', 'RoCEv2', 'AI Networking', 'GPU']
+++

> **Note:** this post is a re-write based on articles found on the internet, cross-checked against NVIDIA's NCCL documentation and published NVLink specifications, and kept here for future reference.

## 1. NCCL: the communication primitives under every training job

Training a model across many GPUs is, mechanically, a loop of compute and synchronization. With 8 GPUs data-parallel training a 7B model, each card holds ~14 GB of bf16 gradients after every backward pass, and all 8 copies must be merged into one identical result on every card — hundreds of thousands of times over a run. The standard tool for this is NVIDIA's **NCCL** (NVIDIA Collective Communications Library). NCCL is not a training framework and not a scheduler; it is the layer that moves tensors between GPUs. Every "data parallel", "tensor parallel", or "parameter sharding" feature in the frameworks above it eventually lands on the small set of primitives this section walks through.

### 1.1 Three words first: rank, communicator, collective

All of NCCL hangs off three terms:

- **rank** — each participating process, usually bound to one GPU. Ranks are numbered 0..P−1.
- **communicator** — a set of ranks plus the connection state between them. A real training job holds *several* communicators at once: the TP group inside a server and the DP group across servers are different communicators over different rank sets (PyTorch surfaces them as process groups).
- **collective** — an operation that every rank in the communicator must participate in. AllReduce is everyone doing AllReduce together; nobody is optional.

And one hard contract that explains most training hangs: **within a single collective call, every rank must pass the same element count and the same datatype, and every rank must actually enter the call.** Violate it and the behavior is undefined — a hang, a crash, or silently wrong numbers. When a training job freezes "in communication", the first check is not a network counter; it is whether some rank skipped the call or passed a mismatched tensor shape.

Notation for the rest of this section: **P** = ranks in the communicator, **N** = elements per shard, **S** = bytes per element.

### 1.2 The eight collectives, in three families

Current NCCL documentation lists eight collectives plus point-to-point send/receive. The names look like a zoo, but they sort into three families by asking two questions: *does the data get reduced (combined arithmetically), and who ends up holding the result?*

> **Version note:** native `ncclGather` / `ncclScatter` / `ncclAlltoAll` APIs are newer additions; older NCCL releases exposed five collectives (AllReduce, Broadcast, Reduce, AllGather, ReduceScatter) and composed the other three from grouped send/recv. Check the installed version before coding against them.

#### Reduction family: AllReduce, Reduce, ReduceScatter

**Reduction** means combining corresponding elements from all ranks with one operator — sum, prod, min, max, avg. The three members differ only in *who keeps the result, and whether it is sliced*:

- **AllReduce** — every rank contributes an equal-size tensor; afterwards **every rank holds the complete reduced result**. The workhorse of data parallelism: PyTorch DDP AllReduces gradient buckets during the backward pass so every replica applies identical updates; tensor parallelism uses it to merge partial matrix products. One property worth stating precisely: AllReduce eliminates the *difference* between model replicas, but saves no memory — every card still holds full parameters, full gradients, and usually full optimizer state. It solves synchronization, not sharding.
- **Reduce** — same arithmetic, but **only the root rank keeps the result** (the root argument is a rank number, not a CUDA device id). For "one place needs the answer": aggregating loss or token counts to rank 0 for logging.
- **ReduceScatter** — reduce everything, then slice the result into P blocks; **rank *i* keeps block *i***. The natural primitive for sharded training (FSDP/ZeRO): if a rank only *owns* 1/P of the parameters, it only needs 1/P of the reduced gradients — gradient memory drops by (P−1)/P. The price is architectural: the training logic above must accept that state lives sharded.

![AllReduce - every rank contributes, every rank receives the full reduced result](/posts/nvlink-nccl-scaleup-scaleout/nccl-allreduce.svg)

![ReduceScatter - reduce all inputs, each rank keeps exactly one shard](/posts/nvlink-nccl-scaleup-scaleout/nccl-reducescatter.svg)

#### Collect-and-distribute family: Broadcast, Scatter, Gather, AllGather

This family does no arithmetic — it only moves data. The four members are distinguished by one question: *where does the complete data start, and where does it end up?*

- **Broadcast** — the root has the full tensor; afterwards everyone has a copy. Used at startup: rank 0 loads or initializes weights, broadcasts them, and every replica starts from identical state.
- **Scatter** — the root holds P chunks and hands chunk *i* to rank *i* — distribution with slicing.
- **Gather** — the reverse: every rank's chunk is collected **to the root only** — for example, pulling scattered eval results onto one card.
- **AllGather** — every rank contributes its shard, and **every rank receives the full concatenation, ordered by rank index**. FSDP's other half: parameters live sharded; just before a layer computes, an AllGather temporarily reassembles the full weights, which are released after use. The cost is just as characteristic: the full tensor materializes on *every* rank, so AllGather timing is exactly the knob FSDP tunes to keep peak VRAM under control.

![AllGather - every rank contributes its shard, every rank ends with the full set](/posts/nvlink-nccl-scaleup-scaleout/nccl-allgather.svg)

A one-line mnemonic for the whole zoo: **"All-" means every rank ends with the final result; "Reduce" means arithmetic happened; "Scatter" means the result was sliced and distributed.** Communication volume in this family is on the order of `(P−1) × N × S` per rank.

#### Full-exchange: AlltoAll

**AlltoAll** is the only true everyone-to-everyone exchange: rank *i* sends its *j*-th chunk to rank *j* and receives one chunk from every peer. Treat the P send buffers as a P×P block matrix and AlltoAll is a distributed transpose. Its main stage is **Mixture-of-Experts**: tokens are routed to whichever rank hosts their expert (dispatch), computed, then routed back (combine) — two AlltoAlls per MoE layer in forward, two more in backward. Three sharp edges: it is data *rearrangement*, not reduction; the fixed-size API is not `AlltoAllV` — real token routing is naturally uneven, so MoE frameworks often build variable-length exchanges from grouped send/recv instead; and it does not support in-place operation — send and receive buffers must be separate, peak user memory `2 × P × N × S`.

![AlltoAll - rank i sends its j-th chunk to rank j, a distributed transpose](/posts/nvlink-nccl-scaleup-scaleout/nccl-alltoall.svg)

#### send/recv: when the pattern is not collective at all

Pipeline parallelism's traffic — stage *k* handing activations to stage *k+1*, gradients flowing back — is point-to-point, and NCCL provides plain **send/receive** for it. Any irregular pattern can be built from it (it is how Gather/Scatter/AlltoAll were composed before they went native), at a cost: who sends, who receives, in what order, with what buffers, and how communication overlaps compute all become your design problem. Collectives are standardized moves; send/recv is choreography you write yourself.

### 1.3 The one equivalence worth memorizing

> **AllReduce ≡ ReduceScatter + AllGather**

First everyone reduces and keeps one slice; then everyone gathers all slices back. Classic **Ring AllReduce** is exactly this decomposition, pipelined around a ring — which is why bandwidth-optimal AllReduce moves about `2(P−1)/P × M` bytes per rank across the links (M = bytes per rank). For the 8-GPU, 14 GB-gradient example: ~24.5 GB through the links per rank per step — *more than the gradient tensor itself*, repeated every step. That is the "communication wall" of large-scale training. And **FSDP is this same identity deliberately split open**, running the two halves at different times so parameters and gradients stay sharded in between:

![FSDP: AllGather parameters to compute, ReduceScatter gradients to shard](/posts/nvlink-nccl-scaleup-scaleout/nccl-fsdp-sequence.svg)

### 1.4 Which parallelism calls which primitive

| Parallelism / framework | Main communication         | When it fires                        | Message profile                                    |
|-------------------------|----------------------------|--------------------------------------|----------------------------------------------------|
| DDP (data parallel)     | AllReduce                  | During backward, per gradient bucket | Large messages, fewer calls — bandwidth-bound      |
| FSDP / ZeRO             | AllGather + ReduceScatter  | Per layer, forward and backward      | Medium messages, many calls — overlap-dependent    |
| Tensor parallel (TP)    | AllReduce / AllGather / RS | Every layer, forward and backward    | Very frequent, latency-critical                    |
| Pipeline parallel (PP)  | send/recv                  | Between adjacent stages              | Point-to-point, overlaps with compute              |
| MoE (expert parallel)   | AlltoAll                   | 2× forward + 2× backward per layer   | Many small uneven messages — latency/balance-bound |

The diagnostic value: when communication is the bottleneck, first find your row. DDP wants big-message bandwidth; MoE wants small-message latency and even load distribution. Those are optimized in opposite directions, and tuning for the wrong row is wasted work.

### 1.5 Memory: three layers before quoting any number

"How much memory does a collective use?" has no single answer — it has three layers:

1. **User buffers** — determined by semantics, exactly computable. AllGather's output is inherently `P×N×S` on every rank; fixed AlltoAll peaks at `2×P×N×S`. The savings lever is **in-place operation**, and NCCL specifies the conditions exactly: AllReduce/Reduce/Broadcast are in-place when `sendbuff == recvbuff`; AllGather when the send buffer sits at `recvbuff + rank × sendcount`; ReduceScatter when the receive buffer sits at `sendbuff + rank × recvcount`; AlltoAll never.
2. **NCCL's internal buffers** — transit buffers per connection (`NCCL_BUFFSIZE`, default 4 MiB), but the true total scales with channel count, connections, protocol, and topology, and every extra communicator (process group) adds a full set. The only accurate accounting is differential: read `cudaMemGetInfo` before/after communicator init and before/after the first collective.
3. **Framework accounting** — PyTorch's `memory_allocated` tracks only its own caching allocator; NCCL allocates outside it and is invisible there. Mixing the two ledgers produces confident wrong conclusions.

Practical summary: the dominant term is almost always layer 1 — AllGather's `P×N×S` grows with model size and communicator width, which is why FSDP limiting how many layers are gathered simultaneously saves more memory than any NCCL knob.

### 1.6 Don't memorize APIs — follow the tensor's fate

The most useful judgment framework: pick (or recognize) a primitive by asking **is this tensor replicated, sharded, or being redistributed?**

- Every GPU has a gradient, every GPU needs the same merged result → **AllReduce**
- Every GPU needs only its own slice of the merged result → **ReduceScatter**
- Every GPU holds a slice, the next computation needs the whole tensor → **AllGather**
- One root hands the same data to everyone → **Broadcast** (sliced per rank: **Scatter**)
- Everyone's data converges on one root → **Reduce** (without arithmetic: **Gather**)
- Every rank exchanges a different slice with every other rank → **AlltoAll**
- The pattern is irregular, or strictly stage-to-stage → **send/recv**

This framing also explains why NCCL can hide the hardware so completely: the application declares data semantics ("AllReduce this"), and NCCL detects the topology — PCIe, NVLink, NVSwitch, InfiniBand, RoCE, plain sockets — and builds rings and trees over whatever it finds, driving communication and computation from the same CUDA kernels. Which sets up the real question for a network engineer: what *is* that hardware underneath — and when does any of this traffic actually reach a network you operate?

## 2. NVLink and NVSwitch: the two domains of an AI cluster

### 2.1 The Scale-Up domain: NVLink and NVSwitch

**NVLink** is NVIDIA's dedicated GPU-to-GPU interconnect, and architecturally it is not "a faster Ethernet" — it is a separate system that bypasses every layer a network engineer normally instruments: not the PCIe bus, not the RDMA NIC (ConnectX-6/7), not the OS network stack, not any switch you manage. Data moves from one GPU's HBM directly into another GPU's HBM; the CPU, the NICs, and your fabric are not on the path.

Inside an H100 SXM server the 8 GPUs are not pairwise-cabled: they interconnect through **4 NVSwitch chips** in a full crossbar. Any two GPUs have a complete-bandwidth NVLink path, so NCCL's collectives see a non-blocking all-to-all topology — no unlucky pairs, no single-link bottleneck, and (with NVSwitch's dedicated buffering) effectively no congestion inside the domain.

![NVLink 900 GB/s - the Scale-Up domain inside one H100 SXM server](/posts/nvlink-nccl-scaleup-scaleout/nvlink-nvswitch-server.svg)

The bandwidth picture, with one correction to the source's comparison table:

| Interconnect          | Bandwidth (per GPU/NIC)    | Convention                 | Scope                   |
|-----------------------|----------------------------|----------------------------|-------------------------|
| NVLink 4.0 (H100 SXM) | 900 GB/s                   | aggregate, both directions | intra-node GPU↔GPU      |
| NVLink 5.0 (GB200)    | 1,800 GB/s                 | aggregate, both directions | intra-node / intra-rack |
| NDR 400G InfiniBand   | ~50 GB/s (~100 GB/s bidir) | per direction (per NIC)    | cross-node              |
| 400GbE Ethernet       | ~50 GB/s (~100 GB/s bidir) | per direction (per NIC)    | cross-node              |
| PCIe Gen5 x16         | ~64 GB/s (~128 GB/s bidir) | per direction              | same host, cross-NUMA   |

> **Fact-check note:** the source table compared NVLink's *bidirectional aggregate* (900 GB/s = 18 links × 25 GB/s × 2 directions) against a NIC's *unidirectional* rate (400 Gb/s = 50 GB/s), which overstates the ratio. Normalized consistently, the honest gap at the H100 generation is roughly **9×** (450 vs ~50 GB/s per direction; 900 vs ~100 GB/s aggregate) — and about double that for Blackwell against the same 400G NIC. The conclusion survives the correction: this is not a gap engineering can close, because you cannot pull NVLink between racks; the physical form factor *is* the boundary.

Latency tells the same story: NVLink end-to-end sits in the sub-microsecond range, while well-tuned cross-node RDMA lands between 1 and 5 µs. For synchronous collectives, latency is additive — every rank waits, and the microseconds compound with scale.

The operational consequence: **when NCCL detects that all GPUs of a communicator share one NVLink domain, it routes the whole collective over NVLink — automatically, no configuration.** Overriding it is possible but pointless: forcing that traffic onto the NIC is a performance regression, not a fix. Inside the 8-GPU domain there is nothing for a network engineer to configure or troubleshoot — an AllReduce there generates zero frames on any wire you own. That is also the trap in single-node benchmarking: if the numbers look wrong, the RoCEv2 config is not the suspect, because the traffic never used it.

### 2.2 The ninth GPU: where Scale-Out begins

The ninth GPU does not fit in the chassis, and from that point part of every collective takes this path:

```text
GPU HBM → PCIe → RDMA NIC (CX7) → optics → leaf/spine switch → optics → RDMA NIC → PCIe → GPU HBM
```

Three structural changes arrive at once:

1. **Latency jumps an order of magnitude.** Sub-µs NVLink becomes 1–5 µs RDMA on every cross-node hop of every synchronous collective.
2. **Per-GPU bandwidth falls ~9×** (450 → ~50 GB/s per direction, using consistent units). The cliff is unavoidable; topology decides how much of the remainder is usable. **Rail-optimized** design uplinks each of the server's 8 NICs to a *different* leaf switch, so 8 GPUs running one AllReduce get 8 independent parallel uplink paths that sum — instead of contending for a single uplink.
3. **Congestion starts existing.** NVSwitch made it a non-issue; an Ethernet fabric carrying AllReduce **incast** is the opposite — many-to-one bursts pile up queues, and without PFC/ECN discipline the result is drops, RDMA go-back-N retransmits, and training throughput falling off a cliff. Everything from my RoCEv2 posts — [PFC/ECN thresholds and headroom](/posts/roce-qos-concepts-and-packet-examples/), [the end-to-end DSCP contract](/posts/rocev2-cisco-cumulus-connectx-end-to-end/) — is the daily work of this domain.

Inside 8 GPUs the network engineer has no seat at the table; beyond 8, every configuration decision lands directly in training throughput.

### 2.3 Hybrid parallelism: mapping section 1's primitives onto the two domains

Real large-model training runs several parallelisms at once, and each has a different traffic personality. This is where section 1 (what each primitive moves) meets this section (what each domain costs):

![Hybrid-parallel traffic paths - TP inside the NVLink domain, PP and DP across the fabric](/posts/nvlink-nccl-scaleup-scaleout/hybrid-parallel-traffic-paths.svg)

- **TP (tensor parallel)** slices the matrix math inside each Transformer layer and communicates dozens of times per training step — AllReduce/AllGather/ReduceScatter at extreme frequency, acutely latency-sensitive. NCCL keeps TP groups on NVLink automatically, and "TP degree ≤ NVLink domain size" is a hard placement rule: a TP group that straddles servers pays network latency an order of magnitude above NVLink on its hottest path. **The misconfiguration signature is worth memorizing:** when topology detection or process binding is wrong and TP silently spills onto the network, training runs far below reference, *nothing* reports an error, and NIC traffic sits much higher than the traffic model predicts. The wrongness is visible only to someone who knows what should never be on the wire at all.
- **PP (pipeline parallel)** hands micro-batch activations between adjacent stages — send/recv, cross-node, moderate volume, deliberately overlapped with compute, so it tolerates fabric latency far better than TP.
- **DP (data parallel)** is the bandwidth story and what Scale-Out fabrics are sized for: every step, every replica AllReduces the full gradient set. A 70B model in mixed precision carries ~140 GB of gradients; at DP=32, each GPU moves `2 × 31/32 × 140 GB ≈ 271 GB` through its links per step — the Ring AllReduce arithmetic from 1.3 at production scale. Multiplied by steps per day, that is the number the capacity plan must survive.
- **MoE** adds AlltoAll — four per MoE layer per step — many small, unevenly sized, latency- and balance-sensitive messages: the hardest pattern for an ECMP fabric to carry well.

### 2.4 What breaks in each domain

**Scale-Up (NVLink) — three problems, none visible from the network side:**

1. **NCCL detects the wrong topology.** Symptom: training >20% below reference on identical hardware, network clean, NIC traffic *higher* than expected. Check:

```text
nvidia-smi topo -m        ! GPU pairs should show NV# (# = NVLink link count); SYS = no NVLink path, NCCL fell back to PCIe/network
```

2. **PCIe/NUMA affinity is wrong.** A typical server has two NUMA nodes, each with 4 GPUs and their NICs; a process bound to the wrong node pays cross-NUMA tax on every host transfer, and throughput drops visibly:

```text
numactl --hardware        ! NUMA topology
lstopo                    ! visualize PCIe/NUMA relationships
```

3. **NVLink runs under its rated bandwidth.** Rare — usually an NCCL/driver version mismatch or BIOS NVLink settings. Watch real NVLink counters with `dcgm-exporter` against the theoretical peak.

**Scale-Out (RoCEv2/IB) — the familiar failure modes, with AI-shaped stakes:**

1. **PFC absent or half-configured.** The classic: PFC configured on the switch, but the NIC trusts PCP while the fabric marks DSCP — PFC never triggers, congestion drops packets, go-back-N retransmits stall the collective. NIC-side verification:

```text
mlnx_qos -i eth0                   ! current QoS/trust configuration
ethtool -S eth0 | grep pause       ! PFC pause counters
```

2. **ECN thresholds unreasonable.** Set too low, normal traffic is constantly throttled; too high, and by the time marking starts the queues are already deep — PFC ends up doing ECN's job. Starting point: mark at ~20–30% of port buffer and tune against measurement; buffer sizes differ enough between switch generations that absolute values do not transfer.
3. **ECMP hash imbalance.** Fabric-wide utilization moderate, but a few links pinned at 100% with drops or PFC storms on exactly those links. Under AllReduce-shaped load, spine per-port utilization should be near-uniform; deviation beyond ~30% means the hash configuration needs work.

### 2.5 The network engineer's core work, by cluster size

- **Single node, 8 GPUs — the baseline.** The network is not the bottleneck; the job is baselining: `nvidia-smi topo -m` showing `NV#` everywhere, `all_reduce_perf` (nccl-tests) compared against NCCL's reference numbers, results recorded as the baseline every later scale step is judged against.
- **8–64 GPUs (2–8 servers).** The network becomes the critical path — the scale where most engineers first *feel* the fabric in training throughput. Rail-optimized topology; RoCEv2 parameters aligned end to end (PFC on the RDMA priority, ECN thresholds, DSCP marking identical on NIC and switch — misalignment shows up as visible jitter in large-batch AllReduce); and an `ib_write_bw` / `ib_send_bw` end-to-end bandwidth validation against link theory *before* any business benchmark runs.
- **64–512 GPUs (8–64 servers).** Real traffic engineering: two-tier spine-leaf or fat-tree with oversubscription no worse than ~2:1 (training forgives far less than inference); ECMP hashing validated under AllReduce-shaped load; on InfiniBand, evaluate Adaptive Routing — congestion-aware path selection worth ~10–20% AllReduce throughput over static ECMP depending on topology and traffic.
- **512+ GPUs.** Every flaw not exposed earlier now amplifies. Adaptive Routing becomes mandatory rather than optional. On InfiniBand, **SHARP** is worth serious evaluation at this scale — it offloads AllReduce aggregation into the switches themselves (Quantum-2 and later, UFM-managed), with measurable gains in latency and bandwidth consumption at thousand-GPU scale. On RoCEv2, **PFC storm protection is a first-class design item**: a fabric-wide PFC storm can stall everything, so configure PFC Watchdog to detect abnormal persistent PFC and automatically break losslessness before the avalanche.

### 2.6 The Scale-Up boundary is moving

The 8-GPU boundary is a property of the H100 generation, not a law. **GB200 NVL72** connects 72 GPUs — 36 Grace-Blackwell superchips, each pairing one Grace CPU with two Blackwell GPUs — into a single rack-scale NVLink 5.0 domain: the rack behaves like one giant accelerator, and the network engineer's entry point moves from "beyond 8 GPUs" to "beyond 72". **NVLink Fusion** pushes further, licensing NVLink attachment to non-NVIDIA silicon.

The durable design rule: **the intervention point is not a fixed GPU count — it is wherever the deployed platform's Scale-Up domain ends.** Inside that boundary, NVSwitch and NCCL own the problem. Outside it, you do.

## 3. First-response toolkit

When training is "slow" and the network is a suspect, in this order:

```text
NCCL_DEBUG=INFO <job>              ! which collective, what sizes, which transport was chosen - answers are usually in here
nvidia-smi topo -m                 ! NV# everywhere it should be
all_reduce_perf -b 8 -e 8G -f 2    ! nccl-tests vs the recorded single-node baseline
ib_write_bw / ib_send_bw           ! end-to-end RDMA bandwidth vs link theory, before blaming the model
dcgm-exporter                      ! NVLink + NIC counters over time, not point samples
```

And the four one-liners this post compresses into: **AllReduce = ReduceScatter + AllGather** (explains Ring AllReduce and FSDP at once); **DDP uses AllReduce, FSDP uses AllGather+ReduceScatter, MoE uses AlltoAll, pipeline uses send/recv**; **memory has three layers** (user buffers have formulas, NCCL buffers need differential measurement, framework counters exclude NCCL); and **a hang means checking that every rank entered the same collective with the same count and datatype** before touching any tuning knob.

## References

- NVIDIA NCCL User Guide — [Collective Operations](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/collectives.html), [In-place Operations](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/usage/inplace.html), [Environment Variables](https://docs.nvidia.com/deeplearning/nccl/user-guide/docs/env.html)
- NVIDIA — [GB200 NVL72](https://www.nvidia.com/en-us/data-center/gb200-nvl72/)
- Related posts on this blog: [RoCE QoS concepts and packet examples](/posts/roce-qos-concepts-and-packet-examples/), [End-to-end RoCEv2: Nexus, Cumulus, ConnectX](/posts/rocev2-cisco-cumulus-connectx-end-to-end/), [RDMA performance tuning](/posts/rdma-performance-tuning/)
