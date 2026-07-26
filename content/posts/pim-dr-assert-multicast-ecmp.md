+++
title = 'PIM DR, Assert, and Multicast ECMP'
date = 2023-11-11T12:00:00+08:00
draft = false
categories = ['Network']
tags = ['PIM', 'Multicast', 'ECMP', 'RPF', 'Routing', 'Network']
+++

Several questions come up whenever two PIM routers share a receiver VLAN over equal-cost paths: if the **DR** and the **Assert winner** are different routers, which one forwards the multicast? Does a single stream ever spread its packets across both paths? If you change nothing, do 500 streams split evenly across the two routers—or does that take configuration? And what changes if the switch becomes a Layer 3 router? This post works through each, with the underlying PIM mechanics and a clear line between what the standards mandate and what is vendor-specific.

## Start here: PIM DR versus PIM Assert

Almost every question below reduces to confusing two PIM mechanisms that sound alike but do different jobs. Getting this pair straight first makes the rest of the post fall into place.

Both are **PIM (Layer 3) control-plane functions**, and both only matter on a **shared multi-access segment**—a LAN or VLAN with two or more PIM routers on it. Neither is an "L2 versus L3" feature, and neither does anything on a point-to-point link. What separates them is *when* and *why* each one acts.

### PIM Designated Router (DR) — the segment's representative

The DR is **elected up front** from PIM Hellos, one per shared segment, before any traffic flows. It acts on behalf of the segment's directly connected hosts:

- **On a receiver segment:** only the DR sends PIM Joins upstream for the segment's receivers (learned from IGMP/MLD)—so the DR is what pulls the streams down onto the VLAN.
- **On a source segment:** the DR registers the source to the RP.

**Election:** highest DR-priority wins, ties broken by highest IP address (default priority 1). It is a control-plane decision, made whether or not any multicast is flowing.

### PIM Assert — the duplicate-stopper

Assert is **reactive and data-plane-triggered**. It fires *only when duplicate multicast packets actually appear on a shared segment* because two routers are both forwarding the same `(S,G)` or `(*,G)` onto it. The routers exchange Assert messages and elect **one forwarder** for that tree; the losers prune that segment from their outgoing list.

**Election (per tree):** a router on the source tree `(S,G)` beats one on the shared tree `(*,G)`; then lower route-preference to the source/RP; then lower metric; then highest IP as the final tiebreak.

Because it needs duplicates on a shared segment, **Assert never fires on a point-to-point link**—which is the real reason people mislabel it a "Layer 2" mechanism.

### When both apply, Assert overrides the DR—for that tree only

If a segment has receivers (so the DR is joining) *and* two routers both end up forwarding the same tree (so Assert fires), the **Assert winner takes over that tree's forwarding and its join responsibility, even if it is not the DR. DR priority does not override the Assert result.** RFC 7761 folds this straight into the forwarding rule: a router forwards for local receivers when

```text
( I_am_DR AND lost_assert == FALSE )  OR  AssertWinner == me
```

So a DR that *loses* an Assert stops forwarding that tree, and a non-DR that *wins* one starts. The catch is that this is **per multicast tree, not a demotion**: the DR stays the DR for the segment—registering sources, handling other groups, resuming if the Assert state times out.

### The two situations, side by side

| | Pure receiver VLAN | Transit / both-forwarding segment |
|---|---|---|
| **DR** | sends the joins, forwards **all** streams | still elected; may or may not be the forwarder |
| **Assert** | **never fires**—only the DR forwards, no duplicates | **fires**—elects one forwarder per tree, overriding the DR |

That left column is the default case for the topology below, and it is why the answers that follow so often reduce to "one router carries everything unless you deliberately change it."

![PIM multicast topology: a sender reaches R1 (the RP), which feeds R2 and R3; both attach to one Layer 2 switch and shared receiver VLAN, with two possible downstream forwarders](/posts/pim-dr-assert-multicast-ecmp/multicast-ecmp-diagram.svg)

## Topology assumption

The answer below assumes that the switch is a **Layer 2 switch** and that R2 and R3 are PIM neighbors on the same receiver-facing VLAN.

The multicast sender is reached through R1, which is the rendezvous point (RP), while the receiver is attached to the shared VLAN below R2 and R3.

## 1. What happens if the PIM DR and Assert winner are different routers?

For the affected multicast tree, the **PIM Assert winner forwards the multicast traffic** onto the receiver VLAN.

PIM DR election and PIM Assert solve different problems:

- The **Designated Router (DR)** represents hosts on a shared LAN. It handles local membership and normally sends the required PIM joins upstream.
- **PIM Assert** resolves duplicate forwarding when more than one upstream router sends the same multicast stream onto a shared LAN.
- The Assert election is maintained separately for the relevant `(*,G)` or `(S,G)` state.
- The Assert winner effectively assumes forwarding and join responsibility for that multicast tree, even if a different router remains the VLAN's DR.

For example:

1. R2 is elected DR for the receiver VLAN.
2. Both R2 and R3 begin forwarding `(S,G)` onto that VLAN.
3. R3 wins the PIM Assert election.
4. R3 continues forwarding `(S,G)`.
5. R2 stops forwarding that stream but can remain the VLAN's DR.

The DR priority does **not** override the Assert result.

### Assert selection

In simplified order, the Assert election considers:

1. Whether the router is using source-tree `(S,G)` state rather than RP-tree state.
2. Unicast route preference toward the source or RP, as applicable.
3. Unicast route metric toward the source or RP.
4. The highest interface IP address as the final tie-breaker.

The downstream router also modifies its upstream choice to use the Assert winner—called the `RPF'` neighbor in the PIM specification—for the affected tree.

## 2. Does one multicast stream use only one ECMP path?

Yes. In conventional PIM forwarding, one multicast tree—normally identified as `(S,G)`—uses **one RPF interface and upstream neighbor at a time**.

Multicast ECMP normally distributes *different multicast trees* across equal-cost paths:

| Multicast tree | Selected RPF path |
|---|---|
| `(S1,G1)` | R2 |
| `(S1,G2)` | R3 |
| `(S2,G1)` | R3 |
| `(S2,G2)` | R2 |

It normally does not divide the packets of a single `(S,G)` stream between R2 and R3. Packet-by-packet splitting could introduce reordering. If both routers sent a complete copy onto the same VLAN, the receiver would instead get duplicate packets—the condition PIM Assert is designed to stop.

## 3. If I change nothing, do 500 streams split 250/250 across R2 and R3?

**No.** With the topology exactly as drawn—a Layer 2 switch and one shared receiver VLAN—leaving the default configuration in place sends **all 500 streams through one router**, not 250 through each.

The reason is the DR's role. Only the elected DR sends PIM joins upstream on behalf of the VLAN's receivers ([RFC 7761, Section 3.1](https://www.rfc-editor.org/rfc/rfc7761.html#section-3.1)), so only the DR pulls the traffic and forwards it onto the VLAN. The other router tracks membership but originates no joins and forwards nothing—a cold standby that takes over only after the DR's Hello holdtime expires (roughly 105 seconds by default) and the state is rebuilt. PIM Assert does not change this: as section 1 showed, Assert elects a single forwarder to remove duplicates; it is not a load balancer.

```text
R2 wins the DR election

500 streams → R1 → R2 → L2 switch → receiver
                       R3 = standby
```

This "one DR carries everything" result is specific to **PIM-SM and PIM-SSM on a receiver-only VLAN**. Two situations change it. PIM-BIDIR ([RFC 5015](https://www.rfc-editor.org/rfc/rfc5015.html)) forwards through a per-RP Designated Forwarder and distributes across routers by design. And if the VLAN is also a transit segment—a downstream router's join is heard on it—or has a directly attached source, then both routers add the outgoing interface and forward, duplicates appear, and Assert elects a per-flow winner by unicast metric. That can *incidentally* split those already-duplicated flows between R2 and R3, but it is a side effect of differing metrics, not a dependable balancer.

### Splitting the streams while keeping the Layer 2 design

To spread the flows across R2 and R3 without changing the shared-VLAN topology, you need **PIM Designated Router Load Balancing (DRLB)**, standardized in [RFC 8775](https://www.rfc-editor.org/rfc/rfc8775.html). The elected DR advertises the candidate routers and a hash; for each flow a **Group Designated Router (GDR)** is chosen by that hash, so forwarding is distributed per flow instead of concentrated on the single DR. Only the routers you want to participate must support and enable it—the ordinary DR election is unchanged, and non-participating routers are simply left out of the candidate set.

One detail matters far more than the "roughly half each" summary suggests, and it is where an expected 250/250 quietly turns into 500/0:

- **The default DRLB hash for ASM is computed on the RP address, not the group.** In a normal single-RP ASM deployment, every group behind that RP hashes to the *same* GDR—so 500 groups can land **entirely on one router, with no balancing at all**. An even split requires **SSM** (which hashes source XOR group), an explicitly zeroed RP hash mask, or groups spread across multiple RPs.
- Even then the split is **statistical, not exact**. A modulo hash over real addresses is only as even as the address distribution, so treat "250/250" as an illustration, never a guarantee.
- The election is **per flow**—`(*,G)` for ASM, `(S,G)` for SSM—not literally "per group."

Note that the ordinary `ip multicast multipath` feature (covered in section 4) does **not** help here. It acts on a router's RPF path selection, and a Layer 2 switch performs no multicast RPF lookup—there is nothing for it to influence. That feature belongs to the Layer 3 redesign below.

## 4. Does making the switch a Layer 3 router change this?

**It gains the *ability* to spread the streams—but doing nothing still won't.** If the switch becomes a Layer 3 last-hop multicast router with separate routed links (ideally separate subnets) to R2 and R3, the picture changes:

![Layer 3 redesign: the switch is replaced by a last-hop multicast router with separate routed links up to R2 and R3, which reach R1 (the RP) and the sender; with multicast multipath enabled the router's RPF/ECMP selection can send different (S,G) trees via R2 or R3, one path per tree, and PIM Assert no longer applies—but by default it still uses a single upstream link](/posts/pim-dr-assert-multicast-ecmp/l3-router-ecmp-diagram.svg)

Here is the catch that trips people up: **doing nothing on the L3 router does not give you 250/250 either.** It performs the RPF lookup, but for equal-cost paths it selects a *single* upstream neighbor for every state—on Cisco, the PIM neighbor with the highest IP address—so all 500 streams ride one link and the other carries none, exactly as the single DR did in section 3. What the Layer 3 design *adds* is the **option** to spread them: cross-path distribution is an opt-in, vendor-specific feature (`ip multicast multipath` and its variants, below), not an emergent property of equal-cost routing.

Once that per-flow hashing is enabled, the router can pick a different upstream neighbor for different multicast trees, distributing distinct `(S,G)` states across R2 and R3. Two properties hold either way, because they come from PIM itself ([RFC 7761](https://www.rfc-editor.org/rfc/rfc7761.html)):

- Each `(S,G)` or `(*,G)` resolves to **one** RPF neighbor, so a single stream always rides **one** path—packets from one flow are never striped across R2 and R3.
- On a path failure, the router re-runs RPF and re-joins the surviving neighbor, moving **only the failed path's** trees. Resilient or consistent hashing, where supported, avoids reshuffling the flows still on the healthy path.

### What it takes to actually distribute the trees

For the 500 streams to spread across R2 and R3, all of the following must hold:

1. The router runs multicast routing and PIM, with routed—not switched—links to R2 and R3, separate subnets preferred.
2. The unicast table contains **equal-cost** routes toward the RP and/or source through both routers, and both next hops pass the RPF check.
3. Multicast multipath hashing is **supported and explicitly enabled**—it is off by default on Cisco.
4. The streams carry varied enough source/group addresses for the hash to spread them.

On Cisco IOS/IOS-XE the feature is `ip multicast multipath`, with a variant ladder:

```text
ip multicast multipath                          ! S-hash: source only
ip multicast multipath s-g-hash basic           ! source + group
ip multicast multipath s-g-hash next-hop-based  ! source + group + next-hop
```

Only the **next-hop-based** variant avoids RPF hash *polarization* on multi-hop paths—with the bare S-hash and even the basic S-G-hash, every router computes the same result from the same inputs, so flows re-converge onto the same path. This CLI is Cisco-specific; NX-OS and other vendors implement multicast ECMP differently. And it remains load **splitting of flows, not balancing of bandwidth**—a single high-rate `(S,G)` still pins its whole load to one link.

### ASM versus SSM changes the path selection

- **ASM** builds the shared tree first: the initial `(*,G)` join follows RPF toward the **RP**, and after the switch to the shortest-path tree the `(S,G)` join follows RPF toward the **source**. Those are two independent lookups against different destinations, so the chosen equal-cost path *can* differ between the two phases—it coincides only if the RP happens to sit on the source-ward path.
- **SSM** builds `(S,G)` state toward the source from the outset, with no RP or shared tree ([RFC 4607](https://www.rfc-editor.org/rfc/rfc4607.html)), so there is no shared-tree-to-SPT transition that could shift the path—though unicast reconvergence can still move it later.

Finally, PIM Assert stops being the selection mechanism in this design. On a shared LAN, Assert elects the single forwarder and, where an Assert winner exists on the RPF interface, it even *overrides* the router's RPF choice. Across **separate routed links** there is no shared segment for duplicates to collide on, so which upstream carries each tree is decided purely by the router's RPF and ECMP selection.

### A single high-bandwidth stream still cannot be split

None of the above combines the bandwidth of R2 and R3 for one large `(S,G)` stream—every mechanism here distributes *flows*, and a single flow always stays on one path. Common alternatives:

- Split the application traffic into multiple multicast groups or sources.
- Use multicast traffic engineering.
- Use a platform-specific load-aware mechanism.
- Use PIM ECMP Redirect ([RFC 6754](https://www.rfc-editor.org/rfc/rfc6754.html)) where supported; it can steer entire multicast trees according to path utilization, but it governs shared multi-access segments and still does not stripe individual packets from one tree across paths.

## References

- [RFC 7761 — Protocol Independent Multicast - Sparse Mode (PIM-SM)](https://www.rfc-editor.org/rfc/rfc7761.html)
- [RFC 8775 — PIM Designated Router Load Balancing (DRLB)](https://www.rfc-editor.org/rfc/rfc8775.html)
- [RFC 4607 — Source-Specific Multicast for IP (SSM)](https://www.rfc-editor.org/rfc/rfc4607.html)
- [RFC 5015 — Bidirectional Protocol Independent Multicast (BIDIR-PIM)](https://www.rfc-editor.org/rfc/rfc5015.html)
- [RFC 6754 — PIM Equal-Cost Multipath (ECMP) Redirect](https://www.rfc-editor.org/rfc/rfc6754.html)
- [Cisco — Load Splitting IP Multicast Traffic over ECMP](https://www.cisco.com/c/en/us/td/docs/ios-xml/ios/ipmulti_optim/configuration/12-2sx/imc-optim-12-2sx-book/imc_load_splt_ecmp.html)
