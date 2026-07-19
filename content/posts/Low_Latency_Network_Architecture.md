+++
title = 'Low_Latency_Network_Architecture'
date = 2022-03-10T20:04:01+08:00
lastmod = 2026-07-19T12:00:00+08:00
draft = false
categories = ['Network']
tags = ['HFT', 'LowLatency_Network']
+++

> Originally published in March 2022. Substantially revised in July 2026: the architecture analysis was rebuilt around current exchange designs (Eurex Co-location 2.0, CME MSGW, Nasdaq NY11-4), the sequencer pattern, traffic-plane segregation, and the hybrid FPGA fast path, with refreshed vendor and latency data. Sources are linked inline; latency figures from vendor datasheets are best-case specs, not independent measurements.

## 1. What a trading system architecture optimizes for

Below is a simple sketch of the key components of an exchange trading system. Legacy systems ran everything on centralized servers; the network carries market data, order routing and clearing.

![Business Flow](/posts/low_latency_network_architecture/low_latency_system.svg)

The lower the latency, the more orders a system can handle — and for a trading firm, the earlier you see information and act on it, the better your edge against other firms.

![Latency on Main US Trading Firms](/posts/low_latency_network_architecture/latency.svg)

> Time measured in microseconds.
>
> Real HFT latency data is confidential; use the above for reference only.

But the 2022 view — "make everything faster" — undersells what this domain actually engineers for. Looking at how exchanges and trading firms build today, three architectural principles matter more than any single device's speed:

1. **Determinism beats raw speed.** A path whose latency you can characterize to a few nanoseconds is worth more than a faster path with unpredictable jitter. Almost every design decision below — one-hop topologies, bufferless fan-out, single-threaded matching — is a determinism decision first and a speed decision second.
2. **Fairness is now a physical design requirement**, not a policy statement. Exchanges engineer equal latency into buildings, cables and switch tiers, and publish timestamps that let participants audit it.
3. **One competitive data plane, several parallel support planes.** Market data, order entry, time distribution and measurement/capture run as physically separate networks, each built from different technology because each carries a different traffic shape.

The rest of this post walks through the architecture at three levels: the exchange side, the trading LAN, and the host/pipeline inside the participant's rack.

## 2. Exchange-side architecture

### 2.1 Fairness by physics

Modern exchange colocation treats latency equality as an engineered product property with four recurring elements:

- **Cable normalization.** Every participant's fiber run is cut or coiled to the same optical length regardless of cabinet position. ICE/NYSE's Mahwah specs state that all LCN/SLC/NMS connections are normalized and verified with Optical Backscatter Reflectometry — even customer-to-customer cross-connects are equalized ([ICE Mahwah technical specs](https://www.ice.com/publicdocs/IGN_Colocation_Mahwah_Technical_Specs.pdf)). Eurex Co-location 2.0 guarantees a maximum deviation of **0.35 m (1.75 ns)** between any two cross-connects to the exchange ([Eurex, Insights into Trading System Dynamics, 2025](https://www.eurex.com/resource/blob/48918/4f724c9415d2731cfb27295db6269c9c/data/presentation-insights-into-trading-system-dynamics.pdf)). The physical constant behind all of this: light in fiber travels roughly 5 ns per meter.
- **Serialized, fan-out-based market data delivery.** Eurex inserted a dedicated **mid-layer switch** (2024) between distribution and access layers specifically so there is exactly *one* serialized feed from the trading backend, replicated to equal-length access switches using Layer-1-style replication — every participant receives the same packet at nanosecond-identical time. Nasdaq's NY11-4 expansion filing (2024) commits that client connections to the matching engine "will be equal across the board" ([Federal Register](https://www.federalregister.gov/documents/2024/09/24/2024-21750/self-regulatory-organizations-the-nasdaq-stock-market-llc-notice-of-filing-and-immediate)).
- **A published serialization point for orders.** CME fronts each of its 17 market segments with a **Market Segment Gateway**: the first complete message to arrive at the MSGW is the first processed by the matching engine ([CME iLink architecture](https://cmegroupclientsite.atlassian.net/wiki/spaces/EPICSANDBOX/pages/457220437/iLink+Architecture)). The race between participants has a single, formally defined finish line. Eurex distinguishes partition-specific gateways (fastest path) from low-frequency gateways (+~12 µs), but routes LF requests *through* the PS gateways so no overtaking is possible — one serialization point regardless of entry path.
- **Latency transparency.** Eurex timestamps every request at the network access layer (White Rabbit-synchronized to ~1 ns), the gateway, and the matching engine, and hands participants a High Precision Timestamp file with the full chain (t_3a → t_9d) so the fairness claims are auditable ([Eurex Insights deck](https://www.eurex.com/resource/blob/48918/4f724c9415d2731cfb27295db6269c9c/data/presentation-insights-into-trading-system-dynamics.pdf)).

### 2.2 The matching engine: partitioning plus the sequencer

The 2022 version of this post described "distributed matching engines" mainly as a scaling trick. The more precise picture is two orthogonal patterns:

**Partitioning by product, never by function.** Exchanges scale horizontally by assigning instrument groups to independent matching-engine instances — CME's 17 market segments, Eurex T7's numbered partitions, NYSE Pillar's symbol sharding across engines. Within a partition, matching stays strictly serial, because price-time priority is meaningless without a total order of events.

**The sequencer pattern.** A single component stamps every input event with a monotonic sequence number and publishes one totally ordered stream (typically UDP multicast). Every downstream component — matcher replicas, market data publishers, drop-copy, surveillance — consumes the *same* stream. Because business logic is a deterministic function of an ordered input stream, state can be rebuilt by replay, and high availability falls out naturally: run identical replicas in lockstep on the same stream. The lineage runs from Island ECN through Nasdaq INET ([how the first low-latency market was designed](https://electronictradinghub.com/how-the-first-true-low-latency-market-was-designed-and-architected/)), through LMAX's single-threaded journaled business-logic processor ([Fowler, The LMAX Architecture](https://martinfowler.com/articles/lmax.html)), to today's packaged form — Raft-replicated deterministic state machines such as [Aeron Cluster](https://aeron.io/docs/cluster-quickstart/replicated-state-machines/), where the sequencer is an elected leader instead of a fixed box.

This is also where time consistency actually gets solved. NTP is nowhere near sufficient; the exchange timestamps events *inside* the ordered stream with hardware clocks, and regulation defines the floor (see section 3.4 for the numbers). PTP remains the distribution workhorse, with White Rabbit at the top end: Eurex piloted White Rabbit in 2018 and made it a permanent sub-nanosecond time service from April 2019 ([Eurex White Rabbit pilot](https://www.eurex.com/ex-en/support/initiatives/archive/high-precision-time-white-rabbit-pilot); [Deutsche Börse time services](https://www.deutsche-boerse.com/dbg-en/markets-services/ps-technology/ps-7-market-technology/ps-n7/ps-connectivity-services-time-services)).

### 2.3 Speed bumps: latency architecture as market design

Some venues deliberately insert delay, and the *shape* of the delay is an architectural choice:

- **Symmetric and physical** — IEX routes all inbound orders through a coiled fiber delay calibrated to exactly 350 µs. When IEX relocated a data center in 2024, the coil grew from 38 to 44 miles to keep the delay invariant ([IEX speed bump](https://www.iex.io/about/speed-bump)).
- **Asymmetric and logical** — Eurex Passive Liquidity Protection defers only *aggressive* (immediately executable) orders by a product-specific time (1–1.5 ms class), while passive quote updates hit the book instantly ([Eurex PLP](https://www.eurex.com/ex-en/trade/market-models/eurex-plp)). The SEC disapproved Cboe EDGA's similar asymmetric proposal in 2020 ([SEC order](https://www.sec.gov/files/rules/sro/cboeedga/2020/34-88261.pdf)) — the regulatory acceptability of delay architecture differs by regime.
- **Batch auctions** ([Budish et al., QJE 2015](https://academic.oup.com/qje/article/130/4/1547/1916146)) would eliminate the race entirely; no major venue has adopted them.

Eurex also manages the FPGA arms race directly in the network: a **Discard IP range** lets firms pre-fire speculative order packets that the access switch silently drops if not needed, and **DSCP flags** in market data headers mark likely trigger packets. Eurex's own telemetry puts the fastest observed market-data-to-order reaction through its infrastructure at about **2.8 µs** ([Eurex Insights deck](https://www.eurex.com/resource/blob/48918/4f724c9415d2731cfb27295db6269c9c/data/presentation-insights-into-trading-system-dynamics.pdf)).

### 2.4 Cloud: the matching engine does not move — the cloud moves to it

Every major exchange-cloud deal converges on the same split-plane shape: the deterministic plane (matching, sequencing, market data origination) stays on dedicated, latency-equalized infrastructure, and the elastic plane (clearing, risk, analytics, distribution) moves to the cloud.

- **Nasdaq + AWS**: AWS Outposts racks were installed *inside* Nasdaq's own Carteret data center, so the "cloud" matching engine keeps the colo network geometry. MRX migrated in December 2022, Nasdaq Bond Exchange in 2023, GEMX in November 2023 (~12 billion daily messages), with roughly 10% round-trip latency *improvement* ([Nasdaq IR](https://ir.nasdaq.com/news-releases/news-release-details/nasdaq-completes-migration-third-us-market-aws)).
- **CME + Google**: clearing and data moved to Google Cloud first (clearing migration was on track to complete around Q1 2026), while Google builds a **private cloud region adjacent to CME's Aurora facility**, contractually designed for equal latency between colo and cloud clients; Globex matching itself is expected to move no earlier than ~2028 and still runs in the existing Aurora building today ([CME/Google announcement](https://www.cmegroup.com/media-room/press-releases/2024/6/26/cme_group_and_googlecloudannouncenewchicagoareaprivatecloudregio.html); [Databento on CME colocation](https://databento.com/blog/cme-colocation)).
- **LSEG + Microsoft**: data platform and analytics to Azure; matching engines untouched ([announcement](https://news.microsoft.com/source/2022/12/12/lseg-and-microsoft-launch-10-year-strategic-partnership-for-next-generation-data-and-analytics-and-cloud-infrastructure-solutions-microsoft-to-make-equity-investment-in-lseg-through-acquisition-of-sh/)).

The academic literature confirms why: cloud networks cannot provide the equalized, deterministic delivery that fairness requires, which is exactly what research like [DBO (SIGCOMM 2023)](https://dl.acm.org/doi/pdf/10.1145/3603269.3604871) tries to reconstruct in software.

## 3. The trading LAN: one plane per traffic shape

The 2022 post called this "independent message bus interface." The fuller statement of the pattern: the trading LAN is not one network but several physically separate planes, each matched to its traffic shape, and the reason they are physical rather than VLANs is microburst isolation and determinism.

![The trading LAN as four physically separate planes](/posts/low_latency_network_architecture/traffic_planes.svg)

### 3.1 Microburst: the phenomenon that shapes the design

Averaged per second, trading traffic looks tiny; zoomed to milliseconds, it saturates links. OPRA bursts exceed 50 Gbps within 1–10 ms windows — typically right after a trade in the underlying, which is exactly when your order flow matters most — and packet rates cross 1 M packets/s ([Databento, Beyond 40 Gbps](https://databento.com/blog/beyond-40-gbps-processing-opra-in-real-time)).

![Microburst](https://songkou.github.io/posts/low_latency_network_architecture/microburst.jpg)

> From an Arista white paper

Because market data bursts hit all subscribers simultaneously and are uncorrelated with a firm's own order flow, any queue shared between market data and orders converts a burst into order-entry jitter at the worst possible moment. Physical plane separation is the isolation mechanism.

### 3.2 Market data plane: one-to-many, bufferless

The dominant distribution technology inside the low-latency perimeter is **Layer 1 replication**, not L2/L3 switching: a crosspoint fans one feed out to every subscriber with no MAC lookup, no forwarding table, and no multicast group state — Arista's 7130 Connect series (the former Metamako line, acquired 2018) does it in **4–6 ns port-to-port** ([Arista 7130 product overview](https://www.arista.com/assets/data/pdf/7130-product-overview.pdf)).

Two things worth noting about multicast:

- Exchange feeds are natively UDP multicast published as **A/B pairs** on independent distribution trees (plus a remote DR feed), and the reconciliation happens *at the host* by sequence-number arbitration — the network stays dumb and fast, and the receiver takes whichever copy arrives first ([Nasdaq feed spec](https://www.nasdaqtrader.com/content/technicalsupport/specifications/dataproducts/MRX_GEMX_DepthofMarket2.02_042024.pdf); [US patent 9,208,523](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9208523)).
- Dynamic multicast machinery (IGMP querying, PIM, OIF walks) is engineered *out* of the fast path — group state converges over CPU-processed control traffic with unpredictable timing, which is a determinism problem. "Joining" an L1 fanout is a configuration decision, not a protocol event. This is the precise, current form of the 2022 post's observation that L1 fan-out guarantees fairness where modular-switch multicast replication cannot.

### 3.3 Order entry plane: many-to-one, bounded contention

The inverse funnel: FPGA multiplexers aggregate many trading servers onto one exchange gateway port. Arista's MetaMux does this in **39 ns average (±7 ns deterministic without contention)** with engineered mux ratios from 4:1 to 48:1 ([MetaMux product brief](https://www.arista.com/assets/data/pdf/ProductBrief-MetaMux.pdf)). The order plane's version of the microburst is **incast** — many servers firing at one gateway on the same trigger — and the architecture accepts it but makes it bounded and observable: contention queues in small input buffers exposed through per-port counters, and the mux ratio is the knob that caps worst-case depth.

Where genuine L2 forwarding is unavoidable, it is FPGA cut-through (Arista SwitchApp, 94–132 ns) rather than a multi-hop fabric. Trading planes are one hop deep by design; spine-leaf topologies, oversubscription, and deep shared buffers belong to the back-office network, which remains a conventional L3 design because its job is reach, not nanoseconds.

### 3.4 Timing plane: out-of-band, and far beyond the regulatory floor

Time is distributed as its own out-of-band plane — GNSS-disciplined grandmaster, PTP over dedicated fiber (plus PPS for hardware), terminating in hardware clocks in NICs, switches and capture devices. Deutsche Börse sells this as two tiers: a PTP service (sub-microsecond) and a White Rabbit service (**sub-nanosecond**) over dedicated cross-connects ([DBAG time services](https://www.deutsche-boerse.com/dbg-en/markets-services/ps-technology/ps-7-market-technology/ps-n7/ps-connectivity-services-time-services)).

The regulatory floors are far looser than what firms deploy:

| Regime | Requirement | Tolerance |
| --- | --- | --- |
| MiFID II RTS 25 (EU, HFT tier) | Divergence from UTC | 100 µs (1 µs granularity) |
| SEC CAT (US, SRO clocks) | Divergence from NIST | 100 µs |
| FINRA 4590 (US, broker-dealers) | Divergence from NIST | 50 ms |

([RTS 25 summary](https://www.emissions-euets.com/time-stamping-and-business-clocks-synchronisation); [SEC CAT clock assessment](https://www.sec.gov/divisions/marketreg/consolidated-audit-trail-clock-synchronization-assessment-051517.pdf); [FINRA notice 16-23](https://www.finra.org/rules-guidance/notices/16-23))

Firms run nanosecond-class sync anyway, because the real job of the timing plane is **latency attribution** — comparing timestamps taken at different physical points — with compliance as a byproduct. This is the mature form of the 2022 post's out-of-band PTP advice: the L1 fanout switches carry OCXO or atomic clock options and PPS in/outs precisely for this role ([Arista 7130 overview](https://www.arista.com/assets/data/pdf/7130-product-overview.pdf)).

### 3.5 Capture plane: the only place deep buffers belong

A parallel passive plane: optical taps on every interesting link → tap aggregation → hardware timestamping (1 ns resolution trailers) → lossless capture. Arista MetaWatch (also ex-Metamako) advertises sub-nanosecond timestamping with 32 GB of deep buffering, marketed explicitly for MiFID II RTS 25 compliance capture ([Arista 7130 overview](https://www.arista.com/assets/data/pdf/7130-product-overview.pdf)). Note the buffer philosophy inversion: bufferless where latency matters, deep buffers where completeness matters — the two never share a device.

The authoritative latency number is **wire-to-wire** (tap timestamp on the ingress market-data fiber vs the egress order fiber); application-level measurement misses NIC, PCIe and stack time and is treated as diagnostic only ([Databento, tick-to-trade](https://databento.com/microstructure/tick-to-trade)).

### 3.6 Serialization and FEC: the 10G/25G question, updated

The 2022 advice — prefer 10G, disable RS-FEC, keep runs short, use passive DAC — has evolved rather than aged. Higher line rate cuts serialization delay proportionally (25/10 = 2.5×), and the FEC penalty that made 25G unattractive (RS-FEC costs roughly 250 ns per hop) is now avoidable: Arista productized FEC-less 25G links using enhanced transceivers, positioning 25G as strictly faster than 10G for trading ([Arista 25G launch, Oct 2023](https://www.arista.com/en/company/news/press-release/18273-pr-20231011)). The architecture lesson is unchanged: at these budgets, physical-layer encoding choices are first-class design decisions.

| Packet size | 64 kb/s | 1 Mb/s | 10 Mb/s | 100 Mb/s | 1 Gb/s | 10 Gb/s |
| --- | --- | --- | --- | --- | --- | --- |
| 64 bytes | 8 ms | 0.512 ms | 51.2 µs | 5.12 µs | 0.512 µs | 51.2 ns |
| 512 bytes | 64 ms | 4.096 ms | 409.6 µs | 40.96 µs | 4.096 µs | 409.6 ns |
| 1500 bytes | 187.5 ms | 12 ms | 1.2 ms | 120 µs | 12 µs | 1.2 µs |
| 9000 bytes | 1125 ms | 72 ms | 7.2 ms | 720 µs | 72 µs | 7.2 µs |

> Serialization delay — the time to clock a frame onto the wire — by frame size and link rate. It falls proportionally with line rate, which is the whole point of the 10G → 25G move. Source: Wikipedia.

## 4. Inside the rack: the tick-to-trade pipeline

### 4.1 The latency ladder

The canonical pipeline is market-data ingest → book build → signal → pre-trade risk → order entry. Where you run each stage defines your class ([Databento, tick-to-trade](https://databento.com/microstructure/tick-to-trade)):

| Architecture | Tick-to-trade |
| --- | --- |
| Standard kernel stack | ~2.7–3.3 µs half-RTT for the network hop alone |
| Tuned software on kernel bypass | ~2 µs floor (dominated by the PCIe round trip) |
| Competitive FPGA/hybrid pipelines | single- to double-digit **nanoseconds** |
| STAC-T0 record (network-I/O trigger loop) | **13.9 ns** actionable latency (Exegy + AMD Alveo UL3524, 2024) |

The STAC-T0 number deserves its caveat: it measures the FPGA's UDP-in/TCP-out trigger loop, not a full strategy ([STAC report](https://docs.stacresearch.com/news/AMD240422)). But it bounds the floor the fast path sits on.

![Tick-to-trade latency ladder](/posts/low_latency_network_architecture/tick_to_trade_ladder.svg)

The kernel-bypass ladder from 2022 is still the right mental model, with updated numbers from AMD's own documentation (X3522-class NICs, 1-byte messages, half-RTT): raw **ef_vi ~797 ns**, **TCPDirect ~830–846 ns**, transparent **Onload ~1.04–1.15 µs**, kernel stack ~2.7–3.3 µs ([AMD UG1586](https://docs.amd.com/r/en-US/ug1586-onload-user/TCPDirect-Latency)).

![Kernel Bypass](/posts/low_latency_network_architecture/kernel_bypass.svg)

### 4.2 The hybrid trigger architecture

The dominant modern design is not "strategy on FPGA" — it is a **trigger architecture**. Software (the slow path) runs the full model: book state, signals, pricing. It continuously *pre-arms* the FPGA or smart NIC (the fast path) with trigger conditions and **pre-computed order templates**. The hardware watches the market-data stream and, on a pattern match, splices the final fields into the canned order and fires — the CPU is off the critical path entirely ([Databento](https://databento.com/microstructure/tick-to-trade); [Algo-Logic tick-to-trade](https://www.algo-logic.com/fpga-tick-to-trade)).

This is the generalization of what the 2022 post described as the Exablaze FDK "hybrid mode" — the pattern won, even as that particular product line ended (see section 5).

![Hybrid trigger architecture](/posts/low_latency_network_architecture/hybrid_trigger.svg)

The trade-offs are structural: hardware iteration is slow (strategy changes need re-synthesis unless expressible as parameterized triggers), on-chip state is limited compared to a software book, and an autonomous firing path needs its own inline risk stage — which is why SEC Rule 15c3-5 pre-trade checks migrated into hardware too: FPGA implementations run 20+ risk checks in ~740 ns, versus 20–100× slower in software ([HFT Review](https://www.hftreview.com/pg/blog/mike/read/7271/how-to-run-20plus-pretrade-risk-checks-in-under-a-microsecond); [Algo-Logic risk check](https://www.algo-logic.com/pre-trade-risk-check)).

### 4.3 Software structure around the bypass

The host pattern is **thread-per-core, shared-nothing, busy-polling**: one pinned thread per isolated core, no shared mutable state, no locks, polling instead of interrupts — protecting tail latency by keeping data in per-core caches ([thread-per-core, ANCS](https://penberg.org/papers/tpc-ancs19.pdf); [Jump Trading's Netdev talk, via LWN](https://lwn.net/Articles/914992/)). The bypass menu maps to needs:

- **Onload / NVIDIA XLIO** — transparent socket acceleration for unmodified applications (XLIO is NVIDIA's current successor to the old Mellanox VMA; [libxlio](https://github.com/Mellanox/libxlio)).
- **TCPDirect** — lower latency, reduced API surface.
- **ef_vi** — raw Ethernet control of the NIC (~10–20 ns receive overhead), for the hand-built critical path.
- **DPDK** — vendor-neutral, chosen for portability and packet rate over the last nanoseconds.
- **AF_XDP** — in-kernel bypass for environments where userspace drivers are impossible (cloud VMs, shared NICs, strict security).

CPU tuning — core isolation, IRQ affinity, cache-line discipline, high-frequency SKUs — remains the constant background work.

### 4.4 Why RoCE and InfiniBand never took over trading

The 2022 post attributed this to "huge modifications on TCP coding." The sharper reason is architectural: **the venue edge dictates the protocol stack.** Exchanges publish UDP multicast market data and TCP order entry over standard Ethernet, so an RDMA fabric buys nothing on the only paths that matter competitively — you must speak the venue's Ethernet protocols anyway. InfiniBand survives inside firms as internal HPC/research-cluster fabric, not as trading connectivity ([Databento, kernel bypass](https://databento.com/microstructure/kernel-bypass); [LWN discussion](https://lwn.net/Articles/914992/)).

## 5. Hardware landscape 2026: what happened to the 2022 vendors

The 2022 post named Cisco Nexus 3548, Arista 7150, "Cisco FusionMux," "Arista MetaMux," Solarflare Onload and Exablaze ExaSOCK. The corrected and updated lineage:

| 2022 reference | What it actually was | Where it is now |
| --- | --- | --- |
| "Arista MetaMux" | Metamako, acquired by Arista 2018 | **Arista 7130 family** — the active flagship line: Connect L1 switches 4–6 ns; MetaMux 39 ns; MetaWatch capture; 7132LB FPGA switch, 150 ns L3 at 25G |
| "Cisco FusionMux" | Exablaze ExaLINK Fusion, acquired by Cisco 2019 | **Cisco Nexus 3550-F** — L1 3–5 ns, FastMux 39–48 ns — but **end-of-sale April 2023 with no replacement**; the Exablaze line is a dead end |
| Cisco Nexus 3548 | Algo Boost ASIC cut-through | Still sold, still sub-200 ns, still 1/10G only — stagnant, no successor named |
| Arista 7150 | ASIC cut-through | Superseded for trading by the FPGA-based 7130/7132 line |
| Solarflare Onload | Solarflare → Xilinx (2019) → **AMD** (2022) | X2522 still sold as "AMD Solarflare"; current flagship X3522; Onload/TCPDirect/ef_vi remain the stack |
| Exablaze ExaSOCK/NICs | Exablaze → Cisco | ExaNIC X25 lived on as Cisco SmartNIC K3P-S (643 ns app-to-app at 10G, vendor figure); future uncertain post-3550 EOL |
| Mellanox VMA | Mellanox → NVIDIA | Succeeded by **XLIO** |

The consolidation is itself an architectural datum: with Cisco's exit, the L1/FPGA-mux tier of exchange colocation has effectively converged on a single vendor's platform.

## 6. Between venues: the WAN mirrors the LAN

The same plane thinking applies between data centers — **RF for signals, fiber for bulk**:

- **Microwave** wins on distance/route directness: ~5 ms saved Chicago–New Jersey versus the fastest fiber; the London–Frankfurt chain measures ~2.3 ms versus ~4.2 ms on fiber ([Journal of Financial Markets, 2023](https://www.sciencedirect.com/science/article/pii/S1386418123000514)). The cost is Mbps-class bandwidth and weather-dependent availability ([IMC 2020 measurement study](https://bdebopam.github.io/papers/imc2020-hft.pdf)).
- **HF shortwave** is the experimental transatlantic frontier — dial-up bandwidth, unbeatable latency ([Sniper in Mahwah](https://sniperinmahwah.wordpress.com/2018/05/07/shortwave-trading-part-i-the-west-chicago-tower-mystery/)).
- **Hollow-core fiber** narrows fiber's latency gap by ~1/3 (light in air vs glass): euNetworks deployed Lumenisity hollow-core in London from 2021, including LSE routes and a 45 km London–Basildon path with 14 km of hollow-core (2022); Microsoft acquired Lumenisity in December 2022 ([euNetworks](https://www.businesswire.com/news/home/20220921005083/en/)).
- Firms run the wireless path as the trigger/signal channel and fiber as the full-data and fallback channel, arbitrated at the receiver by sequence number — exactly like an exchange A/B feed ([US 9,208,523](https://image-ppubs.uspto.gov/dirsearch-public/print/downloadPdf/9208523)).

## 7. Takeaways

1. **Determinism is the product.** L1 fan-out, single-hop planes, single-threaded matchers, bounded incast — every layer trades generality for characterizable latency.
2. **The sequencer is the invariant** of exchange design: one totally ordered stream per partition, deterministic consumers, recovery by replay — whether implemented as INET-style lockstep, LMAX-style journaling, or Raft.
3. **Fairness became physics**: normalized fiber, OBR-verified cross-connects, mid-layer serialization switches, published nanosecond timestamps.
4. **Planes, not networks**: market data, orders, time and capture are separate physical networks with opposite buffer philosophies.
5. **The fast path is a trigger, not a brain**: software thinks, hardware fires, and the mandated risk check rides in the firing path.
6. **The matching engine doesn't go to the cloud; the cloud comes to the matching engine.**

## 8. References

Primary sources are linked inline throughout. The most load-bearing: [Eurex Insights into Trading System Dynamics (2025)](https://www.eurex.com/resource/blob/48918/4f724c9415d2731cfb27295db6269c9c/data/presentation-insights-into-trading-system-dynamics.pdf), [ICE Mahwah colocation specs](https://www.ice.com/publicdocs/IGN_Colocation_Mahwah_Technical_Specs.pdf), [CME iLink/MSGW architecture](https://cmegroupclientsite.atlassian.net/wiki/spaces/EPICSANDBOX/pages/457220437/iLink+Architecture), [Arista 7130 product overview](https://www.arista.com/assets/data/pdf/7130-product-overview.pdf), [Cisco Nexus 3550-F datasheet](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-3550-series/datasheet-c78-743963.html) and [EOL notice](https://www.cisco.com/c/en/us/products/collateral/switches/nexus-3000-series-switches/nexus-3550-f-nexus-3550-h-switches-accessories-eol.html), [AMD UG1586 Onload/TCPDirect latency](https://docs.amd.com/r/en-US/ug1586-onload-user/TCPDirect-Latency), [STAC-T0 AMD/Exegy record](https://docs.stacresearch.com/news/AMD240422), [Databento microstructure guides](https://databento.com/microstructure/tick-to-trade), [Fowler on LMAX](https://martinfowler.com/articles/lmax.html), and the [IMC 2020 microwave measurement study](https://bdebopam.github.io/papers/imc2020-hft.pdf).
