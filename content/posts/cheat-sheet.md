+++
title = 'Cheat_Sheet'
date = 2026-07-21T04:00:00+08:00
draft = false
categories = ['Network']
tags = ['Cheat Sheet', 'SaltStack', 'Arista', 'VXLAN', 'EVPN', 'RDMA', 'Cumulus', 'Cisco Nexus', 'Network']
+++

Every verification and troubleshooting command from the lab posts on this blog — plus a few general-purpose Linux staples — collected in one place and organized **by vendor**, because that's how you reach for a cheat sheet: you're on a box, and you need to know what to run. Each subsection links back to the post it came from where one exists, and explains what each command shows and what a healthy result looks like — a command you can't interpret is just noise.

## 1. SaltStack

Source: [SaltStack Guide](/posts/salt-guide/).

### 1.1 Service health and installation

- `salt --versions-report` — full version dump of Salt and every dependency; the first thing to attach to any bug report, and the quick way to confirm master and minions run compatible versions (master must be same or newer).
- `sudo systemctl status salt-master` / `sudo systemctl status salt-minion` — is the daemon running, since when, and with which PID tree. On modern onedir builds the CGroup lines show the bundled Python (`/opt/saltstack/salt/bin/python3.x`), confirming which interpreter Salt actually uses.
- `sudo journalctl -u salt-minion -f` — live minion log. The classic first-boot failure is right here: `Master hostname: 'salt' not found or not responsive` means the minion can't resolve or reach its master — fix DNS/`master:` config, then restart the service.
- `sudo salt-call --local config.get master` — asks the minion itself which master it is configured to use. `--local` never contacts the master, so it works even when the connection is broken — which is exactly when you need it.
- `sudo salt-master -l debug` / `sudo salt-minion -l debug` — run either daemon in the foreground with debug logging; the fastest way to watch a key exchange or connection attempt fail in real time.

### 1.2 Key management

- `sudo salt-key -L` — lists all minion keys in four buckets: Accepted, Denied, Unaccepted, Rejected. A minion that can reach the master but hasn't been trusted yet sits in Unaccepted.
- `sudo salt-key -a <id>` / `-A` — accept one key / accept everything pending. `-A` is fine in a lab, deliberate in production.
- `sudo salt-key -r <id>` / `-d <id>` — reject / delete a key. After deleting an accepted key, the minion side must also delete its cached master key before the handshake works again.
- `sudo salt-key -f <id>` (master) + `sudo salt-call --local key.finger` (minion) — print the key fingerprint on both ends. Compare them before accepting in production; this is the only defense against a rogue minion registering under a stolen ID.

### 1.3 Connectivity and targeting

- `sudo salt '*' test.ping` — the fundamental liveness check. This is Salt's message-bus ping, not ICMP: `True` means the minion received the job over the bus and answered, proving keys, transport, and the return channel all work.
- `sudo salt '*' cmd.run 'uptime'` — run any shell command everywhere; the general-purpose escape hatch.
- `sudo salt '*' grains.items` / `grains.get os` — dump all facts about each minion / fetch one. Grains are what `-G` targeting matches against.
- Targeting flags, all usable with any command:
  - `-E 'skou_[a-z]+$'` — PCRE regex on minion ID. Anchoring matters: Salt's ID regex matches from the start but not to the end, so `web` matches `web01` *and* `website`; add `$` for an exact end.
  - `-P 'os:Ubuntu.*'` — regex on a grain value.
  - `-G 'os:Ubuntu'` — exact grain match.
  - `-L 'web01,db01'` — explicit list.
  - `-I 'role:webserver'` — pillar match.
  - `-C 'E@web.* and G@os:Ubuntu'` — compound; `E@`/`G@`/`P@`/`I@` prefixes select the matcher per clause.
- `sudo salt-key -L` (inventory role) — the authoritative list of every node Salt manages: all minions under `Accepted Keys`, whether currently online or not. A powered-off minion still appears here.
- `sudo salt-run manage.status` — that inventory split into `up` and `down` in one output: which accepted minions answer the bus and which don't. Scope it with the usual targeting: `manage.status tgt='skou*'`.
- `sudo salt-run manage.up` / `manage.down` — only the responsive / only the unresponsive minions; faster than `test.ping` on a big fleet because they don't wait out the timeout for dead nodes.
- `sudo salt-run jobs.active` — currently running jobs; where to look when a command seems hung.

### 1.4 States and the top file

- `sudo salt '<id>' state.show_top` — what the state top file assigns to this minion. Empty when nothing matches — check the target line and match type.
- `sudo salt '<id>' state.highstate test=True` — dry run: reports what *would* change without changing anything. Caveat learned the hard way: a dry run cannot validate a state that bootstraps its own dependencies, because the bootstrap never executes in test mode.
- `sudo salt '<id>' state.highstate` (or bare `state.apply`) — apply everything the top file assigns. `state.apply <name>` instead applies one named state directly, *bypassing* the top file. Remember the two targeting layers: the CLI ID scopes who runs a highstate now; the top file decides what each minion gets.
- `sudo salt '<id>' sys.list_state_functions <module>` — asks the minion's loader whether a state module exists right now (e.g. `mysql_database` → expect `.present`/`.absent`). Empty output = the module isn't loadable: missing dependency, or (on Salt 3008) a module moved out of core into a `saltext-*` extension.
- `sudo salt '<id>' saltutil.refresh_modules` / `saltutil.sync_all` — ask the minion to rebuild its module cache / re-sync custom modules from the master. When a freshly installed dependency still isn't seen, restart `salt-minion` — a fresh process rebuilds the loader unconditionally.
- `sudo salt-call --local <fn> -l debug` — run a function in a brand-new process with a brand-new loader, printing why modules were skipped (`'mysql' __virtual__ returned False: <reason>`). Two traps: it probes the machine you type it on, and it runs **without pillar**, so pillar-dependent behavior (like connection settings) differs from a master-driven run.

### 1.5 Pillar

The pillar debugging funnel, from "where does Salt look" down to "what did the minion get" — all run on the master:

- `sudo salt-run config.get pillar_roots` — the directory the master searches (default `/srv/pillar`). The runner form reads the *master* config; `salt-call --local config.get pillar_roots` would read the minion's own config instead.
- `sudo salt-run pillar.show_top minion=<id>` — which pillar SLS files the *pillar* top file assigns to that minion. Empty = the pillar top file isn't matching (missing `/srv/pillar/top.sls`, assignment placed in the state top file by mistake, tab in the YAML).
- `sudo salt-run pillar.show_pillar <id>` — the fully rendered pillar, computed master-side with no minion involved.
- `sudo salt '<id>' saltutil.refresh_pillar` then `pillar.items` — push fresh pillar to the minion, then read the minion's cached copy. A bare `----------` from `pillar.items` means the master rendered zero data for this minion — and the `True` from `refresh_pillar` only confirms the minion asked, not that anything matched.

State-side equivalents for comparison: `file_roots` ↔ `pillar_roots`, `state.show_top` ↔ `pillar.show_top`, `state.show_highstate` ↔ `pillar.show_pillar`. The pillar commands are runners (`salt-run`) because pillar renders on the master; states render on the minion.

### 1.6 Application-level checks (MySQL example)

- `sudo salt '<id>' pkg.version mysql-server` / `service.status mysql` — package installed at which version; service running.
- `sudo salt '<id>' mysql.db_list` / `mysql.db_tables <db>` / `mysql.user_list` — query MySQL through Salt's execution module. If these return `not available`, the minion's Salt Python lacks PyMySQL (`salt-pip install pymysql`); if they work but the `mysql_*` *states* don't exist, install `saltext-mysql` (Salt 3008 moved the state modules out of core) and restart the minion.

## 2. Arista EOS

### 2.1 RoCE, QoS, PFC and ECN

Source: [Arista EOS RoCEv2 Lossless Config](/posts/arista-eos-roce-config/). During a traffic test, verify in this order: classification first, then ECN, then PFC, then the watchdog — a packet mapped to the wrong queue cannot be fixed by changing ECN thresholds.

- `show qos maps` — the operational DSCP→traffic-class and traffic-class→queue maps. Verify RoCE data (e.g. DSCP 26) lands in the intended queue and CNP (e.g. DSCP 48) in its strict-priority queue *before* debugging anything downstream. Watch for the EOS low-end inversion: DSCP 0–7 map to TC 1, DSCP 8–15 to TC 0.
- `show qos profile <NAME>` / `show qos profile summary` — the profile's contents and where it is attached. A spine accidentally running the leaf profile shows up here (or in the ECN thresholds below).
- `show qos interfaces ethernet <n>` / `... trust` — per-interface QoS state and the operational trust mode. Routed ports default to DSCP trust; switched ports default to CoS — a profile copied to the wrong port type classifies everything wrong.
- `show qos interfaces ethernet <n> ecn` — the programmed ECN min/max thresholds per queue. This is also the leaf-vs-spine sanity check: each device must show *its own* thresholds (e.g. leaf 256/512 KB, spine 512/768 KB).
- `show qos interfaces ethernet <n> ecn counters queue` — actual ECN marking counters (needs `random-detect ecn count` under the interface queue, and on some platforms `hardware counter feature ecn out`). Marking activity here is the *first* congestion signal in a healthy DCQCN loop.
- `show priority-flow-control` — PFC enabled/active per port and for which priorities. Expect only the no-drop priority (3 in the reference design); CNP and control queues are deliberately not no-drop.
- `show priority-flow-control counters` — pause frames sent/received. These are rates to watch, not pass/fail values: a few pauses during bursts are normal; continuously climbing TxPFC after ECN is active means thresholds are too high or the sender isn't reacting to CNPs.
- `show priority-flow-control status` / `show priority-flow-control counters watchdog` — watchdog state and stuck-queue events. The watchdog only monitors PFC queues that also have `bandwidth guaranteed` configured.
- `show dcbx ethernet <n>` — whether IEEE DCBX is enabled and active and which TLVs the peer sent. Don't hunt for a literal "negotiated" state — and remember static PFC works with DCBX disabled entirely.
- `show interfaces ethernet <n> counters queue detail` — per-queue drops. Drops on the no-drop queue mean the lossless chain is broken somewhere.

### 2.2 VXLAN, EVPN, MLAG and multicast

Source: [Arista VXLAN EVPN Lab](/posts/arista-vxlan-bum-her-vs-multicast/).

#### Underlay

- `show ip ospf neighbor` — every fabric P2P link should be FULL. With `ip ospf network point-to-point` there is no DR/BDR noise.
- `show ip route <vtep-loopback>` — on a spine, the MLAG pair's shared VTEP /32 must show **two ECMP next-hops**; that ECMP is what makes the anycast VTEP and its failover work.
- `ping <remote-vtep> source Loopback1` — VTEP-to-VTEP reachability with the *actual* VXLAN source address. A plain ping can succeed while the sourced one fails (missing loopback advertisement), which breaks the overlay invisibly.

#### MLAG

- `show mlag` — expect `State: Active`, `negotiation status: Connected`, peer-link Up. Also shows the MLAG system ID used as the shared router MAC.
- `show mlag interfaces detail` — per-MLAG-port state; `active-full` means both peers agree the port is dual-connected.
- `show port-channel <n> detailed` — LACP membership; `PeerEthernetX` entries confirm the partner side.
- `show mlag config-sanity` — peer-consistency check for the MLAG config itself.

#### VXLAN data plane

- `show interfaces vxlan 1` — the single richest output: source interface **and the IP it resolved to**, VLAN-VNI map, flood mode and its source (`CLI` = static, `EVPN` = IMET-built), the HER flood list, `Static VRF to VNI mapping` (symmetric IRB), the auto-allocated dynamic L3VNI VLAN, and `MLAG Shared Router MAC` (all-zeros = `vxlan virtual-router encapsulation mac-address mlag-system-id` not configured). Two classic finds here: `Source interface is Loopback0` when the design says Loopback1 — remote VTEPs are flooding to an address this switch never decapsulates; and `Replication/Flood Mode is not initialized yet` — no working BUM mechanism (on vEOS-lab, the signature of the unsupported multicast flood mode).
- `show vxlan vtep` — remote VTEPs the data plane currently knows, with tunnel types (`unicast, flood`).
- `show vxlan flood vtep` — the BUM flood list per VLAN. Static mode shows what you typed; EVPN mode shows what Type-3 IMET routes built — the list that should update itself when a VTEP joins.
- `show vxlan address-table` — remote MACs and which VTEP each one sits behind. Empty on both ends of a stretched VLAN after traffic = nothing is crossing; check the source-interface mismatch first.
- `show mac address-table vlan <n>` — local view: remote MACs should appear against port `Vx1`.
- `show vxlan vni` — every VNI including the VRF-mapped L3 VNI.
- `show vxlan config-sanity detail` — the one-shot consistency check: VTEP IP, VLAN-VNI map, flood list, routing (the `Virtual VTEP IP is not configured` FAIL belongs to the legacy flood-and-learn direct-routing model — with EVPN symmetric IRB, judge by the VRF route table and a ping instead), and the MLAG peer-consistency block. Worth running after every change.
- `show ip virtual-router` — the anycast gateway virtual MAC and which virtual IPs are active.

#### EVPN control plane

- `show bgp evpn summary` — every leaf should hold an Established session to each spine RR with prefixes exchanged. `PfxRcd 0` on an Established session usually means `send-community extended` is missing — routes arrive without RTs and are silently unimportable.
- `show bgp evpn route-type imet` — Type-3 routes: one per (VTEP, VNI). These build the HER flood lists; an MLAG pair advertises them from both peers with the shared VTEP as next-hop.
- `show bgp evpn route-type mac-ip` — Type-2 routes: host MACs, plus IPs once ARP'd. MAC+IP entries are what power ARP suppression.
- `show bgp evpn route-type ip-prefix` — Type-5 routes: tenant prefixes carrying the L3 VNI and Router-MAC extended community. Present only with symmetric IRB; their absence *is* the asymmetric/symmetric discriminator.
- `show bgp evpn route-type imet detail` — with a multicast underlay, the PMSI tunnel attribute here names the group per VNI.

#### Multicast underlay

- `show ip pim neighbor` — PIM adjacency per fabric link. All up proves the control plane.
- `show ip pim rp` — which RP this router uses for which groups.
- `show ip mroute` — the actual `(*,G)`/`(S,G)` state. The lab's diagnostic signature: PIM neighbors up + RP known + mroute **permanently empty** + VXLAN flood mode never initialized = platform limitation (vEOS-lab has no VXLAN multicast data plane), not misconfiguration.

#### Inter-VLAN routing (IRB)

- `show ip interface brief | include Vlan` — every SVI that should route must be up/up. EOS autostate keeps an SVI down until the VLAN has a forwarding member port; a down SVI silently blackholes its whole subnet. `no autostate` is the explicit fix.
- `show vrf` / `show ip route vrf <TENANT>` — tenant VRF exists with routing enabled; local connected subnets plus remote subnets learned via the L3 VNI.
- `show ip arp vrf <TENANT>` — resolved host bindings in the tenant VRF.
- The **TTL trick** — no switch access needed: ping between VLANs and read the TTL at the destination. One decrement (63 from a 64 start) = asymmetric IRB (routed once, at ingress); two decrements (62) = symmetric IRB (routed at ingress and egress). Spines never affect the count — they route only the outer header.

## 3. Cisco NX-OS

### 3.1 VXLAN EVPN and Multi-Site

Source: [VXLAN EVPN Architecture](/posts/vxlan-evpn-architecture/). The disciplined order: underlay/MTU → VTEP loopback reachability → BGP EVPN sessions → Type-3 (before debugging BUM) → Type-2 (endpoints) → Type-5 (prefixes) → mappings (VLAN-VNI, VRF-L3VNI, RD/RT) → NVE peers and MAC/ARP tables → packet capture.

- `show bgp l2vpn evpn summary` — overlay session state per neighbor. Established with prefixes is the baseline; a route can be *received* here and still not forwarded — BGP receipt is only stage one of installation (RT import, encapsulation, next-hop resolution, and hardware programming all follow).
- `show bgp l2vpn evpn` / `route-type 2` / `route-type 3` / `route-type 5` — the EVPN table by route type. Missing Type-2 = host never learned or export/import RT mismatch; missing Type-3 = no BUM replication list for that VNI; Type-5 present but not installed = overlay-index recursion failed (gateway IP / router MAC / ESI unresolvable).
- `show nve peers` — the VTEPs this switch has actually formed tunnels with; the data-plane counterpart of the Type-3 table.
- `show nve vni` — VNI state, replication mode (ingress-replication vs mcast group), and the associated VRF for L3 VNIs.
- `show nve multisite fabric-links` / `dci-links` — Multi-Site tracking state on a BGW; a BGW that lost one side stops being a valid transit and these show why.
- `show mac address-table dynamic` — bridged endpoint learning; remote MACs point at the NVE interface.
- `show ip arp vrf <name>` — tenant-VRF ARP; feeds suppression and symmetric IRB.
- `show forwarding route vrf <name>` — what was actually programmed into hardware — the final word when BGP and RIB look right but traffic drops.

## 4. Cumulus Linux (NVUE + FRR)

Source: [VXLAN EVPN Cumulus Lab](/posts/vxlan-evpn-cumulus-5.4-lab-guide/), plus the Cumulus multicast section of [VXLAN EVPN Architecture](/posts/vxlan-evpn-architecture/). Cumulus splits visibility between NVUE (`nv show ...`, the configuration system's view) and FRR (`sudo vtysh -c "..."`, the routing daemon's view) — use both.

### 4.1 NVUE and configuration state

- `nv config diff` / `nv config apply` / `nv config save` — pending changes, apply, persist. Inspect the diff before applying onto a box with existing config.
- `nv config show > backup.txt` — full config export; do it before any change.
- `stty cols 200` before wide `nv show` outputs — NVUE truncates columns with "…" on narrow consoles (an EVE-NG HTML5 console artifact, not an error).

### 4.2 MLAG

- `nv show mlag` — the paired view of applied vs operational values. Key reads: `peer-alive True`; `anycast-ip` appearing only in the *operational* column (clagd activates the shared VTEP address only after the VXLAN consistency check passes — the moment it also lands on `lo` and enters BGP via `redistribute connected`); `backup-active True` for the underlay keepalive.
- `clagctl` — the classic view: roles, peer link, VxLAN Anycast IP, system MAC, and the per-interface CLAG table. Any `Proto-Down Reason` other than `-` on a bond or the vxlan device is a peer-consistency failure.
- `nv show interface bond1` — the MLAG bond's operational state on each peer.

### 4.3 BGP and EVPN

- `sudo vtysh -c "show bgp summary"` — all address families per neighbor. Unnumbered sessions display as `leaf1(swp1)`; a bare interface name with `AS 0` and state `Active` is a session that never established. One TCP session carries both IPv4 and EVPN — identical uptimes across the blocks confirm it.
- `sudo vtysh -c "show bgp l2vpn evpn summary"` — the EVPN block alone. Sanity-check the prefix math: a spine's `PfxSnt` should equal the sum of what all leaves sent it (it reflects everything, imports nothing).
- `sudo vtysh -c "show bgp l2vpn evpn route"` — the full EVPN table; add `route type macip` / `type multicast` / `type prefix` to filter Type-2/3/5.
- `sudo vtysh -c "show bgp l2vpn evpn route type prefix"` — Type-5 check specifically. `No EVPN prefixes (of requested type) exist` with tenant routing configured means the tenant VRF's BGP block is missing one of its two required lines: `redistribute connected` (puts SVI subnets into the tenant table) *and* `route-export to-evpn` (exports them as Type-5).
- `sudo vtysh -c "show bgp vrf TENANT1 ipv4 unicast"` — the tenant VRF's own BGP table. `View/Vrf ... is unknown` on a spine is *correct* — spines carry no tenant state.

### 4.4 VXLAN and EVPN state

- `sudo vtysh -c "show evpn vni"` — every VNI with type (L2/L3), MAC/ARP counts, remote VTEP count, and tenant VRF. Empty on a spine is by design.
- `nv show nve vxlan` — VTEP settings applied vs operational: source address, MLAG shared address, `arp-nd-suppress`, and flooding mode (`head-end-replication evpn`).
- `ip -d link show type vxlan` — the kernel's view of the VXLAN device; useful when NVUE and reality seem to disagree.
- `ip route show vrf <name>` / `ip route get <ip>` — kernel routing per VRF; which path a specific destination takes.

### 4.5 Multicast underlay (PIM/MSDP, when used instead of HER)

- `nv show vrf default router pim` / `nv show interface <swp> router pim` — PIM config state per NVUE.
- `sudo vtysh -c 'show ip pim neighbor'` / `'show ip pim rp-info'` — adjacencies and RP mapping.
- `sudo vtysh -c 'show ip msdp peer'` / `'show ip msdp sa'` — the Anycast-RP mesh state and active sources (Cumulus requires a full MSDP mesh between RPs).
- `sudo vtysh -c 'show ip mroute'` — verify both `(*,G)` and `(S,G)`, their RPF incoming interface, and the outgoing lists; then capture on a fabric link and confirm outer destination = the configured group, UDP 4789, expected VNI.

## 5. Linux hosts

### 5.1 RDMA and perftest

Source: [Perftest RDMA Benchmarking](/posts/perftest/). All tests are client–server: run the same command and options on both machines, adding the server's IP on the client. Firewall: TCP 18515 (parameter negotiation) and UDP 4791 (RoCEv2 data).

**Stack inspection:**

- `ibv_devices` — RDMA devices present (e.g. `rxe0` for Soft-RoCE, `mlx5_0` for ConnectX). An empty table means no RDMA stack — on a lab VM, load Soft-RoCE: `sudo modprobe rdma_rxe && sudo rdma link add rxe0 type rxe netdev <if>` (does not survive reboot).
- `ibv_devinfo` — port state and parameters; want `state: PORT_ACTIVE`, `link_layer: Ethernet` for RoCE, and `active_mtu 4096` on tuned hardware.
- `ibv_devinfo -v | grep -i atomic` — `ATOMIC_HCA` means the device supports remote atomics, prerequisite for the `ib_atomic_*` tests.
- `rdma link show` — kernel view of RDMA links and their netdev bindings.
- `show_gids` (MLNX_OFED) — GID table with types. Pick the **RoCEv2 GID carrying your IPv4 address** and pass its index with `-x`; the wrong GID index is the classic "ping works, RDMA fails".

**The benchmarks:**

- `ib_write_bw` / `ib_read_bw` / `ib_send_bw` — bandwidth for the three verbs (one-sided write, one-sided read, two-sided send). Quote **BW average**, not peak; a wide peak/average gap means an unstable run. perftest's "MB/sec" is MiB/s — use `--report_gbits` for line-rate comparisons.
- `ib_write_lat` / `ib_read_lat` / `ib_send_lat` — latency with small messages. Quote **t_typical**, not t_avg (the mean is dragged up by outliers), and read the p99/p99.9 columns for jitter. The tools time a round trip and report half as the one-way estimate.
- `ib_atomic_bw` / `ib_atomic_lat` with `-A FETCH_AND_ADD|CMP_AND_SWAP` — remote atomic rate/latency. Judge by **MsgRate** (ops/s), not bandwidth — every op is 8 bytes.
- Flags that matter: `-d <dev>` device, `-s <size>` message size, `-n <iters>` count, `-D <sec>` duration mode (`--cpu_util` adds CPU cost), `-a` sweep 2 B–8 MiB (shows where the link saturates), `-R` connect via RDMA-CM (required for `-T <tos>` DSCP marking), `-x <gid>` GID index, `-q <n>` queue pairs (saturating very fast links, ECMP entropy), `-b` bidirectional (must be on both ends), `-F` silence the CPU-governor warning, `--out_json` machine-readable output.

**Host tuning checks:**

- `cpupower frequency-set -g performance` — the CPU governor perftest warns about.
- `cat /sys/class/infiniband/<dev>/device/numa_node` then `numactl --cpunodebind=<n> --membind=<n> <test>` — keep the benchmark on the NIC's NUMA node; crossing sockets costs real bandwidth.
- `lspci -vv` (look at `LnkSta`) — PCIe width/generation; a 400G NIC in a x8 slot cannot deliver.
- MTU end to end: host 9000, switch 9216; `ibv_devinfo` should show `active_mtu 4096`.

### 5.2 Host networking (bonding, MTU, throughput)

Source: host sections of the [VXLAN EVPN Cumulus Lab](/posts/vxlan-evpn-cumulus-5.4-lab-guide/).

- `ip -br address` / `ip route` — addresses and routing; on multi-homed lab hosts confirm exactly **one** default route (management), with specific `ip route add` entries for fabric subnets.
- `cat /proc/net/bonding/bond0` — LACP truth: want `Number of ports: 2`, a nonzero Actor Key, the same Aggregator ID on both slaves, and the **Partner Mac Address equal to the switch pair's MLAG system MAC** — the proof the host sees two switches as one LACP partner. Slaves stuck at `Speed: Unknown / Duplex: Unknown` in EVE-NG mean the `virtio-net-pci` NIC model — switch the node to `e1000`.
- `ethtool <if>` / `ethtool -i <if>` — link speed/duplex and driver; what the bonding driver reads to admit a port into an 802.3ad aggregator.
- `ip neigh show <gw>` — the gateway's resolved MAC; against an anycast gateway it must be the fabric-wide virtual MAC, identical on every leaf.
- `ping -M do -s 1472 -c 3 <ip>` — path-MTU probe with DF set: 1472 + 28 = a 1500-byte packet that must survive VXLAN's ~50-byte overhead on a 9216-MTU fabric.
- `iperf3 -s` / `iperf3 -c <ip> -P 4 -t 30` — sustained multi-stream throughput across the overlay; parallel streams also exercise ECMP hashing.

### 5.3 DNS lookups (dig)

Reading any `dig` output, three lines matter beyond the answer itself: the **ANSWER SECTION** (records, with the second column being the TTL — seconds the record may stay cached; watch it count down across repeated queries to confirm you're hitting a cache), the **SERVER** line at the bottom (which resolver *actually* answered — the fastest way to catch a query going to a different resolver than you assumed), and **status** in the header (`NOERROR` vs `NXDOMAIN` = name doesn't exist vs `SERVFAIL` = resolution broke upstream).

- `dig <name>` — query through the system resolver. Multiple A records in the answer (google.com returns six) is normal load distribution, not an error.
- `dig @8.8.8.8 <name>` — query a *specific* DNS server, bypassing the local resolver. Comparing this against the plain query is the standard test for a stale cache, split-horizon DNS, or a hijacking middlebox: same name, different answers, and the SERVER lines tell you who said what.
- `dig <name> ANY` — ask for all record types at once (A, AAAA, NS, MX...). Useful overview, but many public resolvers now answer ANY with a minimal or refused response (RFC 8482) — query the specific type (`dig <name> AAAA`, `dig <name> NS`) when ANY comes back thin.
- `dig <name> +noall +answer` — suppress everything except the answer section; the script- and screenshot-friendly form. `dig <name> +short` goes further and prints only the values.
- `dig <name> +trace` — resolve iteratively from the root servers down, printing each delegation step. This shows where in the chain resolution breaks — root, TLD, or the domain's own nameservers — instead of just telling you it failed.
- `dig -x <ip>` — reverse (PTR) lookup: who does this IP claim to be? The query is really for `<reversed-ip>.in-addr.arpa`, which is why the answer can differ from — or not exist despite — the forward record.
- A `CNAME` in the answer (e.g. `www.facebook.com → star-mini.c10r.facebook.com`) means the name is an alias; resolution continues at the target, and the final A/AAAA records belong to the target, not the alias.

### 5.4 Listening ports and sockets

Who is listening on what, answered from four angles — socket table, process table, single port, and from the outside:

- `sudo ss -tulpn` — the modern one-liner: `-t` TCP, `-u` UDP, `-l` listening, `-p` owning process, `-n` numeric (no DNS/service-name lookups). Add `-w` for raw sockets. Two reading notes: the **Local Address** column tells you the reachability scope — `127.0.0.1:port` is loopback-only (invisible from outside no matter what the firewall says), `0.0.0.0`/`*` is all IPv4 interfaces, `[::]` all IPv6; and don't pipe this through `grep LISTEN` when `-u` is included — UDP is connectionless, its sockets show `UNCONN`, so the grep silently deletes every UDP line.
- `sudo netstat -tulpn` — the legacy equivalent (net-tools package); same flags, same output shape, on boxes without `ss`. On **macOS**, netstat takes different flags: `netstat -anp tcp` / `netstat -anp udp` (there `-p` selects the protocol, not the process).
- `sudo lsof -i -P -n | grep LISTEN` — the process-first view: every process with network sockets. `-P`/`-n` keep ports and addresses numeric.
- `sudo lsof -i:22` — everything touching one specific port, both the listener and current connections; the quickest "what exactly is on this port?" answer.
- `sudo nmap -sTU -O <ip>` — the outside view from another host: scan TCP (`-sT`) and UDP (`-sU`) and fingerprint the OS (`-O`). This is the ground truth for what's *reachable* — a service can listen locally yet be filtered by a firewall in between, and only an external scan shows that difference. Scan only hosts you're authorized to test.

### 5.5 Changing an IP address permanently (per distro)

First, the distinction that causes most "it broke after reboot" tickets: `ip address add 10.10.80.91/24 dev ens33` and `ip route add default via 10.10.80.1` change the **running** state only — perfect for testing, gone at reboot. Permanent means editing the distro's network configuration and letting its network stack apply it. Which file and which restart command depend on the distro *and generation*.

**List all interfaces (any distro):**

- `ip -br link` — every interface, one line each: name, admin/oper state, MAC. The fastest inventory.
- `ip -br address` — the same list with IP addresses; the two `-br` commands together answer "what interfaces exist and what do they carry".
- `ip link show <if>` / `ip -s link show <if>` — full detail for one interface; `-s` adds RX/TX counters and error/drop columns (climbing errors = physical-layer problem).
- Reading the state flags: `UP` = admin-enabled with link; `DOWN` = administratively disabled; `UP` combined with `NO-CARRIER` = enabled but no link — cable, peer port, or (in EVE-NG) the connector. Admin state and carrier are independent, exactly like a switch port's `shutdown` vs line-protocol.

**Shut / unshut an interface (the `shutdown` / `no shutdown` equivalent):**

- `sudo ip link set <if> down` / `sudo ip link set <if> up` — runtime admin down/up, any distro. This is what the failover tests in the lab posts use (`ip link set eth0 down` to break one bond member). Runtime only: state reverts to the config at reboot — and note that `down` on an ifupdown/netplan interface does not remove its addresses from the config, just stops the interface.
- ifupdown distros (Debian/Alpine): `sudo ifdown <if>` / `sudo ifup <if>` — these also deconfigure/reconfigure the addresses from `/etc/network/interfaces`, so they double as a per-interface config reapply.
- NetworkManager distros (RHEL 8/9): `sudo nmcli device disconnect <if>` / `connect <if>` — keeps NM's view consistent; a plain `ip link set down` can be silently re-upped by NM, which is why "the interface won't stay down" happens on those boxes.
- Permanent down: ifupdown — remove the `auto <if>` line; netplan — set the interface `optional: true` and remove its addresses, or `activation-mode: off` on newer releases; NetworkManager — `nmcli connection modify <profile> connection.autoconnect no`.

The permanent-IP configuration itself:

**Ubuntu 18.04+ — netplan.** Config lives in `/etc/netplan/*.yaml` (names vary: `00-installer-config.yaml`, `50-cloud-init.yaml`):

```yaml
network:
  version: 2
  ethernets:
    ens33:
      dhcp4: false
      addresses: [10.10.80.91/24]
      routes:
        - to: default
          via: 10.10.80.1
      nameservers:
        addresses: [8.8.8.8, 1.1.1.1]
```

- `sudo netplan try` — applies with an automatic 120-second rollback unless you confirm; **always use this over SSH**, it's the difference between a typo and a locked-out box.
- `sudo netplan apply` — apply immediately (console sessions, or after `try` confirmed).
- Notes: modern netplan uses `routes: - to: default` (the old `gateway4:` is deprecated); YAML is indentation-sensitive, spaces only. On cloud images, `50-cloud-init.yaml` is regenerated by cloud-init — to make edits stick, disable its network module: `echo 'network: {config: disabled}' | sudo tee /etc/cloud/cloud.cfg.d/99-disable-network-config.cfg`.

**Debian / older Ubuntu / Alpine — ifupdown.** Config is `/etc/network/interfaces`:

```text
auto ens33
iface ens33 inet static
    address 10.10.80.91/24
    gateway 10.10.80.1
```

- Apply on Debian: `sudo systemctl restart networking`, or per-interface `sudo ifdown ens33 && sudo ifup ens33` (safer — only touches the one interface).
- Apply on Alpine/OpenRC (the EVE-NG lab hosts): `rc-service networking restart`.

**RHEL / CentOS 7 — ifcfg files.** One file per interface at `/etc/sysconfig/network-scripts/ifcfg-eth0`:

```text
DEVICE=eth0
BOOTPROTO=none
ONBOOT=yes
IPADDR=10.10.80.91
PREFIX=24
GATEWAY=10.10.80.1
DNS1=8.8.8.8
```

- Apply: `sudo systemctl restart network` (RHEL/CentOS 7 only — this service no longer exists on 8/9). `BOOTPROTO=none` + `ONBOOT=yes` is the static-at-boot combination; `dhcp` there re-enables DHCP.

**RHEL / CentOS / Rocky / Alma 8 and 9 — NetworkManager (nmcli).** The ifcfg format is deprecated (RHEL 9 stores connections as keyfiles under `/etc/NetworkManager/system-connections/`); edit through nmcli instead of the files:

```bash
sudo nmcli connection modify eth0 ipv4.method manual \
  ipv4.addresses 10.10.80.91/24 ipv4.gateway 10.10.80.1 ipv4.dns 8.8.8.8
sudo nmcli connection up eth0
```

- `nmcli connection show` lists connection profiles (the profile name is not always the device name — check first). `nmcli connection reload` re-reads files if you did edit them by hand. Changes made with `connection modify` are persistent by definition — they edit the stored profile, and `connection up` applies it.

**All distros — verify, then reboot-test:**

- `ip -br address` / `ip route` — the address is on the interface and exactly one default route points where intended.
- `hostnamectl set-hostname <name>` — the permanent-hostname counterpart (systemd distros), for when the renumbering comes with a rename.
- DNS check: `resolvectl status` on systemd-resolved distros (Ubuntu), `cat /etc/resolv.conf` elsewhere — confirms the nameservers from the config actually landed.
- The only real proof of "permanent": `sudo reboot`, then `ip -br address` again. A config that survives a service restart can still lose to cloud-init or NetworkManager overwriting it at boot.

### 5.6 Managing services (systemd)

Ubuntu, Debian, RHEL/CentOS/Rocky/Alma all use systemd today, so one command set covers them. Keep the two state axes separate in your head: **active** (running right now?) and **enabled** (starts at boot?) — a service can be any combination of the two, and "it worked until the reboot" is exactly an *active-but-disabled* service.

**Check one service:**

- `systemctl status <svc>` — the full picture: active/inactive/failed, enabled/disabled, uptime, PID tree, and the last few log lines. The one command that answers "is it running and why not".
- `systemctl is-active <svc>` / `is-enabled <svc>` / `is-failed <svc>` — single-word answers, ideal in scripts and quick checks.
- `sudo journalctl -u <svc> -f` — follow the service's log live; `-n 100` for the last 100 lines, `--since "10 min ago"` to scope. Where you go when `status` shows failed and the last-lines snippet isn't enough.

**Start / stop / restart:**

- `sudo systemctl restart <svc>` — full stop-then-start; picks up config changes, drops connections. What we used on `salt-minion` throughout the Salt guide.
- `sudo systemctl reload <svc>` — re-read config *without* dropping connections, for services that support it (nginx, sshd); `reload-or-restart` falls back automatically for ones that don't.
- `sudo systemctl start <svc>` / `stop <svc>` — one-shot, no boot-time effect.
- `sudo systemctl enable --now <svc>` — enable at boot *and* start immediately, in one command (`disable --now` is the mirror). Plain `enable`/`disable` only changes the boot behavior.
- `sudo systemctl daemon-reload` — re-read unit files after you edit one; required before a restart honors unit-file changes, and the fix for "changed the unit file but nothing changed".

**List services:**

- `systemctl list-units --type=service` — all services *loaded right now* with their state; add `--state=running` for only the active ones, `--state=failed` (or the shortcut `systemctl --failed`) for the broken ones — the first command to run on a box that "came back weird" after a reboot.
- `systemctl list-unit-files --type=service` — every service *installed*, with its boot-time setting (enabled/disabled/static/masked). This is the list that answers "what starts at boot", which `list-units` does not.

**Non-systemd stragglers:** Alpine/OpenRC (the EVE-NG lab hosts) uses `rc-service <svc> status|start|restart`, `rc-status` to list, and `rc-update add <svc>` for boot enablement. The old `service <svc> status` wrapper still works on most distros and delegates to systemd — fine interactively, but scripts should use `systemctl` directly for its exit codes.

### 5.7 Firewall rules (ufw, firewalld, nftables/iptables)

Same split as network config: Ubuntu fronts the kernel firewall with **ufw**, the RHEL family with **firewalld**, and both are only front-ends — the kernel enforces nftables/iptables, which is where the ground truth lives. Golden rule before touching any of them over SSH: **make sure the rule allowing your own SSH session exists before enabling or reloading.**

**Ubuntu — ufw:**

- `sudo ufw status` — active state and rules. `Status: inactive` means **nothing is enforced**, even if rules have been added — they're staged until `sudo ufw enable`.
- `sudo ufw status numbered` — the same list with an index per rule; the view you need for deleting. `status verbose` adds default policies and logging level.
- `sudo ufw allow 4505:4506/tcp` — open a port range; ufw requires the `/tcp` (or `/udp`) suffix whenever a range uses the colon. Single ports and service names also work: `sudo ufw allow 22/tcp`, `sudo ufw allow OpenSSH`.
- `sudo ufw allow from 10.10.80.0/24 to any port 4505:4506 proto tcp` — source-scoped rule; better practice than opening a port to the world (this is the Salt master example from the [SaltStack guide](/posts/salt-guide/)).
- `sudo ufw delete 3` — delete by number from the `numbered` view (indexes shift after each delete — re-run `status numbered` between deletes). Deleting by rule text also works: `sudo ufw delete allow 22/tcp`.
- Modify = delete + re-add; ufw has no in-place edit. When order matters (a deny that must beat a broad allow), place it explicitly: `sudo ufw insert 1 deny from <ip>` — rules match top-down.
- `sudo ufw reload` — reapply after edits; `sudo ufw enable` / `disable` turn enforcement on/off (enable persists across reboots).

**RHEL / CentOS / Rocky / Alma — firewalld:**

- `sudo firewall-cmd --state` — running or not. `sudo firewall-cmd --list-all` — the active zone's full ruleset (services, ports, sources); add `--zone=<z>` for others.
- The **runtime vs permanent** split is the firewalld trap: without `--permanent` a change applies instantly but dies at the next reload/reboot; with `--permanent` it survives but does **nothing until** `--reload`. So either run the command twice (with and without `--permanent`), or apply runtime-only and then persist what works: `sudo firewall-cmd --runtime-to-permanent`.
- `sudo firewall-cmd --permanent --add-port=4505-4506/tcp` — open a port range (note firewalld uses `-` in ranges where ufw uses `:`). Services by name: `--add-service=https`.
- Source-scoped: `sudo firewall-cmd --permanent --add-rich-rule='rule family=ipv4 source address=10.10.80.0/24 port port=4505-4506 protocol=tcp accept'`.
- Remove is the mirror image: `--permanent --remove-port=4505-4506/tcp` / `--remove-service=...` / `--remove-rich-rule='...'`, then `--reload`.
- `sudo firewall-cmd --reload` — reapply the permanent config **without breaking established connections**; this is the normal "restart". `sudo systemctl restart firewalld` is the heavier full restart — rarely needed, and it briefly interrupts.

**The kernel layer — nftables / iptables (any distro):**

- `sudo nft list ruleset` — everything actually loaded in the kernel, including rules ufw/firewalld/Docker/Salt injected behind your back. When the front-end says one thing and traffic does another, this is the arbiter.
- `sudo iptables -L -n -v --line-numbers` — the legacy view with packet/byte counters (a rule whose counters never move isn't matching) and line numbers for deletion.
- Raw edits: `sudo iptables -I INPUT 1 -p tcp --dport 22 -j ACCEPT` (insert at top), `-A` (append at bottom — order matters, first match wins), `-D INPUT <num>` (delete by number).
- Persistence caveat: raw `iptables`/`nft` edits vanish at reboot unless saved — `iptables-save` + the `netfilter-persistent`/`iptables-services` package, or write `/etc/nftables.conf` and manage it via `sudo systemctl restart nftables`. If ufw or firewalld is in charge, make the change through *it* instead of racing it at the kernel layer.

### 5.8 Shell and text-processing staples

The commands that turn raw output into answers. Most of their value is in pipelines, so each entry ends with a realistic one-liner from network-ops life.

- `grep` — filter lines by pattern. The flags that matter: `-i` case-insensitive, `-v` invert (everything *except*), `-E` extended regex, `-r` recurse a directory, `-c` count matches, `-o` print only the matched part, `-A/-B/-C <n>` show lines after/before/around a match — the context flags are what make log reading bearable.
  `grep -B 2 -A 5 'ERROR' /var/log/salt/minion` — each error with its lead-up and aftermath.
  `grep -oE '([0-9]{1,3}\.){3}[0-9]{1,3}' file | sort -u` — extract every unique IPv4 address from anything.
- `awk` — column processor. `$1..$n` are whitespace-split fields, `$NF` the last one, `-F` changes the delimiter, and it can do arithmetic across lines.
  `ss -tn | awk '{print $5}'` — peer address column only.
  `awk -F: '{print $1}' /etc/passwd` — usernames.
  `awk '{sum+=$2} END {print sum}' file` — total a column.
- `sed` — stream editor; 90% of its use is substitution. `sed 's/old/new/g'` per line, `-i` edits the file in place (`-i.bak` keeps a backup — cheap insurance), `sed -n '20,40p'` prints a line range, `sed '/^#/d'` deletes matches.
  `sed -n '/interface Vxlan1/,/!/p' running-config` — print just one config stanza, the switch-config trick worth memorizing.
  `grep -v '^#' file | grep -v '^$'` — the config file without comments and blank lines (pure grep alternative: works everywhere).
- `sort` — `-n` numeric (without it, 10 sorts before 9), `-r` reverse, `-u` unique, `-k <n>` sort by field n, `-t` field delimiter, `-h` human sizes (2K < 1G).
  `sort -t . -k1,1n -k2,2n -k3,3n -k4,4n` — sort IPv4 addresses correctly octet by octet.
- `uniq` — collapse *adjacent* duplicates, which is why it's almost always `sort | uniq`. `-c` prefixes counts — the heart of the most useful pipeline in operations:
  `awk '{print $1}' access.log | sort | uniq -c | sort -rn | head` — top talkers in any log, top-N of anything.
- `tee` — split a pipe: pass output through *and* write it to a file. `-a` appends. The essential trick: `echo 'cfg line' | sudo tee -a /etc/some.conf` — the shell redirect `sudo echo >> file` fails because the *redirect* runs as you, not root; `tee` runs under sudo, so it works (this is exactly how the Salt repo key was installed in the guide).
  `some_long_test | tee run1.log` — watch live and keep the evidence.
- `head` / `tail` — first/last `-n <n>` lines. `tail -f` follows a growing file; `tail -F` survives log rotation — prefer it for daemon logs.
- `wc -l` — count lines; the instant "how many" after any filter: `grep -c` when it's one pattern, `| wc -l` when it's a pipeline.
- `cut` — simple column extraction when awk is overkill: `cut -d: -f1 /etc/passwd`. Falls apart on variable whitespace — that's awk's territory.
- `tr` — character translate/delete: `tr 'A-Z' 'a-z'` lowercases, `tr -d '\r'` strips Windows line endings (the fix for `bad interpreter: /bin/bash^M`), `tr -s ' '` squeezes repeated spaces so `cut` can cope.
- `xargs` — turn stdin into arguments: `grep -rl 'old-vlan' /srv/salt | xargs sed -i 's/old-vlan/new-vlan/g'` — edit every file that matched. `-n 1` one-per-invocation, `-P 4` parallel.
- `find` — locate by predicate, then act: `find /var/log -name '*.log' -mtime -1` (modified in the last day), `-size +100M`, `-exec <cmd> {} \;` to run something on each hit.
- `column -t` — align any whitespace output into readable columns; `watch -n 1 '<cmd>'` — rerun a command every second, full-screen: `watch -n 1 'ip -s link show eth0'` is a poor man's interface counter dashboard.
- Pipeline glue worth remembering: `cmd1 && cmd2` (run 2 only on success), `cmd 2>&1 | grep ...` (include stderr in the pipe — without it, error messages sail past your grep), `$(cmd)` (substitute output), `!!` and `sudo !!` (repeat last command — the classic forgot-sudo recovery).
