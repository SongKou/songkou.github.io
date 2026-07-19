+++
title = 'VXLAN EVPN Architecture'
date = 2026-07-19T12:00:00+08:00
draft = false
categories = ['Network']
tags = ['VXLAN', 'EVPN', 'BGP', 'Cisco Nexus', 'Data Center', 'Multi-Site', 'Network']
+++

VXLAN EVPN combines a scalable Layer 2 data plane with a standards-based MP-BGP control plane. VXLAN carries Ethernet frames across a routed IP fabric; EVPN distributes endpoint, subnet, and tunnel-reachability information so the fabric does not have to discover everything by flooding.

{{< toc title="Contents — 22 sections" closed="true" >}}

## Standards and implementation scope

VXLAN EVPN is not defined by one document. The architecture is assembled from a data-plane encapsulation, an EVPN control plane, a mapping between EVPN and network-virtualization overlays, and later IRB and prefix-route extensions:

| Document | Role in the architecture |
|---|---|
| [RFC 7348 - VXLAN](https://www.rfc-editor.org/rfc/rfc7348.html) | VXLAN frame format, VNI, VTEP behavior, UDP transport, MTU, and basic flood-and-learn operation |
| [RFC 7432 - EVPN](https://www.rfc-editor.org/rfc/rfc7432.html) | EVPN NLRI, route types 1-4, multihoming, MAC mobility, split horizon, and designated-forwarder procedures |
| [RFC 8365 - EVPN for NVO](https://www.rfc-editor.org/rfc/rfc8365.html) | Applies EVPN to VXLAN and other IP-overlay encapsulations; defines how VNIs and tunnel attributes are carried |
| [RFC 9135 - EVPN IRB](https://www.rfc-editor.org/rfc/rfc9135.html) | Symmetric and asymmetric integrated routing and bridging procedures |
| [RFC 9136 - EVPN IP Prefix](https://www.rfc-editor.org/rfc/rfc9136.html) | Route Type 5 and overlay-index resolution for IP prefixes |

Cisco configuration examples in this article should be read alongside the [current Nexus 9000 VXLAN configuration guide](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x.html). Platform, line-card, topology, and release restrictions can be more important than the nominal CLI syntax.

## 1. Why overlays exist

Modern data centers need workload mobility, multi-tenancy, elastic scale, and automation. A traditional Layer 2 design makes the physical network carry too much endpoint state and stretches failure domains as the network grows. An overlay separates two concerns:

- The **underlay** provides resilient IP reachability between tunnel endpoints.
- The **overlay** provides tenant-facing Layer 2 or Layer 3 services over those IP paths.

This separation lets the fabric core concentrate on high-capacity IP forwarding. Endpoint state is moved toward the leaf switches, where hosts actually attach. Multiple tenants can share the same physical network while keeping independent address spaces and policies.

### 1.1 Layer 2 and Layer 3 overlays

A Layer 2 overlay transports complete Ethernet frames. It emulates a LAN segment and therefore supports IP and non-IP payloads, but it also carries the familiar Layer 2 concerns of broadcast, unknown-unicast, and multicast traffic. VXLAN, OTV, and VPLS are examples.

A Layer 3 overlay transports IP packets and abstracts routed connectivity. It gives mobility independently of a single subnet and contains Layer 2 flooding, but it cannot transparently carry arbitrary Ethernet semantics. LISP, IPsec-based overlays, and L3VPNs are examples.

VXLAN is primarily a Layer 2 overlay encapsulation, but VXLAN EVPN fabrics commonly provide both bridging and routing through integrated routing and bridging (IRB).

### 1.2 Problems VXLAN addresses

Classic VLANs use a 12-bit identifier, giving about 4,000 usable segments. VXLAN uses a 24-bit VXLAN Network Identifier (VNI), providing approximately 16 million segments. The larger space is well suited to large multi-tenant environments.

VXLAN also removes the requirement that Layer 2 adjacency must remain inside one physical switched domain. It encapsulates an Ethernet frame in UDP/IP, allowing the frame to cross routed boundaries and use ECMP paths in the transport network.

## 2. VXLAN encapsulation and VTEPs

VXLAN is often described as **MAC-in-UDP**. The original Ethernet frame becomes the payload of an outer UDP/IP packet:

```text
Outer Ethernet
  Outer IP
    UDP (destination port 4789)
      VXLAN header (contains the 24-bit VNI)
        Original Ethernet frame
          Original payload
```

![VXLAN packet format and UDP destination port 4789](/posts/vxlan-evpn-architecture/vxlan-packet-format.svg)

The outer source and destination IP addresses identify the source and destination VTEPs. The inner addresses identify the original communicating endpoints. This distinction is fundamental during troubleshooting: the underlay forwards the outer packet, while the overlay service interprets the inner frame.

### 2.1 Header fields and encapsulation overhead

The base VXLAN header is eight bytes. Its most important fields are:

- The **I flag** indicates that the VNI field is valid and must be set for ordinary VXLAN traffic.
- The **VNI** is 24 bits, providing 16,777,216 possible values before implementation reservations.
- Reserved bits are transmitted as zero and ignored on receipt.
- UDP destination port **4789** is the IANA-assigned default. Early products used other ports, so the destination port can be configurable.
- The UDP source port should be derived from inner-packet fields. This creates flow entropy for ECMP while keeping packets from one flow on a stable path. RFC 7348 recommends the dynamic/private range 49152-65535. [RFC 7348, Section 5](https://www.rfc-editor.org/rfc/rfc7348.html#section-5)

For an IPv4 underlay without optional headers, VXLAN adds 50 bytes before the outer FCS: 14 bytes outer Ethernet, 20 bytes IPv4, 8 bytes UDP, and 8 bytes VXLAN. An IPv6 underlay adds 70 bytes before optional extension headers. A fabric transporting a 1500-byte inner Ethernet frame therefore commonly uses a physical MTU of at least 1550 bytes for IPv4, with operational headroom often rounded higher.

VTEPs must not fragment VXLAN packets. Intermediate IPv4 routers technically can fragment them, but a receiving VTEP may discard fragments. The reliable design is an end-to-end underlay MTU that accommodates the complete encapsulated frame and validation with both ordinary and DF-bit test traffic. [RFC 7348, Section 4](https://www.rfc-editor.org/rfc/rfc7348.html#section-4)

### 2.2 Virtual Tunnel End Point

A **VTEP** (Virtual Tunnel End Point) originates and terminates VXLAN tunnels. A hardware VTEP on a leaf switch has two logical sides:

- A local-facing bridge or routed interface connects servers, hypervisors, firewalls, and other services.
- A fabric-facing IP interface, normally a loopback, identifies the VTEP in the underlay and supplies the outer VXLAN source address.

The same function can run in a hypervisor virtual switch. A physical switch that maps traditional VLANs to VNIs acts as a VXLAN gateway.

### 2.3 VLAN, bridge domain, and VNI

On a leaf, a local VLAN or bridge domain is mapped to an L2 VNI. The VLAN identifier is locally significant; the VNI is the overlay-wide segment identifier. Although deployments often use an easy-to-read mapping such as VLAN 100 to VNI 10100, the values do not have to be mathematically related.

| Object | Scope | Function |
|---|---|---|
| VLAN/bridge domain | Local to a switch or attachment domain | Connects local interfaces into a Layer 2 segment |
| L2 VNI | Overlay-wide | Identifies a bridged tenant segment across VTEPs |
| VRF | Tenant routing domain | Provides an independent Layer 3 routing table |
| L3 VNI | Overlay-wide, normally one per VRF | Carries routed traffic for symmetric IRB |

## 3. Underlay design

The underlay must provide IP reachability between every participating VTEP. It can use OSPF, IS-IS, EIGRP, or BGP. The course emphasizes that all proven IP-routing practices still apply: redundant links, fast convergence, predictable addressing, and ECMP.

A leaf-spine Clos fabric is common because every leaf is the same number of routed hops from every other leaf. With equal-cost routes through multiple spines, VXLAN flows can be distributed across the fabric. The UDP source port is usually derived from a hash of the inner packet, giving the underlay entropy for ECMP.

Operational requirements include:

- Unique and reachable VTEP loopbacks.
- An MTU large enough for the original frame plus roughly 50 bytes of VXLAN/UDP/IP overhead.
- Consistent routing and ECMP behavior.
- Multicast routing only when underlay multicast replication is selected.
- Failure detection and convergence fast enough for the service objective.

The underlay should know VTEP loopbacks, not tenant prefixes. Tenant reachability belongs to the overlay control plane.

### 3.1 Underlay routing choices

There is no EVPN requirement to use one particular underlay protocol. Common choices are:

| Underlay | Strengths | Design cautions |
|---|---|---|
| OSPF or IS-IS | Familiar link-state behavior, fast convergence, clear separation from overlay BGP | Requires address and area/level planning; large fabrics need disciplined summarization and flooding scope |
| eBGP | Simple failure domains, natural leaf/spine policy boundaries, strong operational visibility | ASN plan, next-hop handling, multipath, and maximum-path settings must be consistent |
| BGP unnumbered | Minimizes point-to-point IPv4 addressing and works naturally with leaf/spine links | Depends on IPv6 link-local transport and platform support; troubleshooting skills must cover both address families |

Regardless of protocol, verify that every VTEP loopback has equal-cost reachability through the intended spines. A healthy BGP EVPN session does not prove that the data-plane VTEP next hop is reachable at the required MTU.

### 3.2 ECMP behavior

The outer five-tuple gives the underlay a routable flow key. The outer source and destination IP addresses remain the VTEP pair, so the varying UDP source port supplies most of the per-flow entropy. This normally balances many tenant flows across ECMP paths but keeps one flow ordered. A polarization problem can still occur if platforms use identical hash inputs or if traffic contains too little inner diversity; validate real workload distributions rather than assuming all links will be equal.

### 3.3 Multicast underlay requirements

When L2 VNIs use multicast replication, the underlay must also provide:

- PIM adjacency on the appropriate routed links.
- Rendezvous-point design for ASM, or a valid SSM design when supported.
- Correct VNI-to-group mapping and acceptable group scale.
- IGMP/MLD and multicast route-state observability.
- Consistent RPF reachability to each VTEP source.

Ingress replication removes these multicast dependencies but transfers replication work and bandwidth to the ingress VTEP. The right choice depends on BUM rate, VTEP count, hardware replication capacity, and operational familiarity—not merely on which configuration is shorter.

## 4. Design Considerations

### 4.1 iBGP overlay with an IGP underlay versus eBGP everywhere

The BGP decision occurs at two distinct layers:

- The **underlay** must advertise or otherwise resolve VTEP loopbacks.
- The **overlay** uses the `l2vpn evpn` address family to carry endpoint, VNI-membership, multihoming, and IP-prefix routes.

Two widely deployed models are an iBGP EVPN overlay above an OSPF/IS-IS underlay and an “eBGP everywhere” fabric inspired by [RFC 7938](https://www.rfc-editor.org/rfc/rfc7938.html). RFC 7938 describes eBGP as the stand-alone routing protocol for a large-scale Clos underlay; applying eBGP to the EVPN overlay as well is an implementation design layered on top of that model.

![Comparison of iBGP EVPN over an IGP underlay and eBGP everywhere](/posts/vxlan-evpn-architecture/bgp-design-comparison.svg)

#### Model A: IGP underlay and iBGP EVPN overlay

OSPF or IS-IS advertises the infrastructure loopbacks. Leaves establish iBGP EVPN sessions with redundant spines or dedicated nodes acting as route reflectors.

This model has several convenient properties:

- iBGP route reflection normally preserves the originating leaf's BGP next hop, so the EVPN next hop remains the actual VTEP loopback.
- A common fabric ASN makes automatic RT derivation such as `ASN:VNI` consistent on every leaf.
- Route reflectors do not need tenant VRFs, SVIs, or VNIs. Their job is to reflect EVPN NLRI and its extended communities.
- The IGP has one narrow responsibility: infrastructure reachability and ECMP to loopbacks.

Its costs are a two-protocol operating model, redundant route-reflector design, and the scaling and flooding characteristics of the selected IGP. The shared ASN also provides less natural hop-by-hop policy separation than an eBGP Clos.

#### Model B: eBGP everywhere

In this design, BGP IPv4 unicast supplies the underlay and eBGP EVPN sessions supply the overlay. A common ASN scheme assigns one AS to the spine tier and a unique AS per leaf, leaf pair, or rack. It creates clear failure and policy boundaries and removes the need for iBGP route reflection.

The default eBGP behavior requires deliberate overlay policy:

1. **Preserve the VTEP next hop.** When a spine advertises an eBGP EVPN route to another leaf, ordinary eBGP next-hop processing would make the spine the next hop. The overlay route must retain the originating VTEP address. On NX-OS, Cisco documents a route map using `set ip next-hop unchanged` on the spine's outbound EVPN sessions.
2. **Retain EVPN routes without local VNIs.** A transit spine has no tenant VRFs or import RTs. On NX-OS, `retain route-target all` under `address-family l2vpn evpn` allows it to retain and advertise EVPN routes that have no locally importable RT. [Cisco Nexus 9000 eBGP EVPN procedure](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/m_configuring_vxlan_bgp_evpn.html)
3. **Make RTs consistent across leaf ASNs.** If automatic RTs are derived as `local-AS:VNI`, unique leaf ASNs produce different RTs for the same VNI. Use explicit fabric-wide RTs or a documented feature such as NX-OS `rewrite-evpn-rt-asn` where supported.
4. **Handle the AS path intentionally.** Reusing an ASN across multiple leaves or a redundant leaf pair can trigger eBGP loop prevention. Depending on topology and vendor, designs may require `disable-peer-as-check`, `allowas-in`, `as-override`, or a different ASN allocation. These commands solve different problems and should not be substituted blindly.
5. **Carry extended communities.** EVPN import policy depends on Route Targets, so overlay peers must propagate the required standard and extended communities.

A representative NX-OS spine pattern is:

```text
route-map NEXT-HOP-UNCH permit 10
  set ip next-hop unchanged

router bgp 65000
  address-family l2vpn evpn
    retain route-target all

  neighbor 10.255.0.11
    remote-as 65101
    update-source loopback0
    ebgp-multihop 2
    address-family l2vpn evpn
      send-community
      send-community extended
      route-map NEXT-HOP-UNCH out
```

This is a structural example, not a universal template. Whether sessions use directly connected addresses or loopbacks, whether `ebgp-multihop` is needed, and the exact community/peer-AS commands depend on platform and design.

#### Operational comparison

| Consideration | IGP + iBGP EVPN | eBGP everywhere |
|---|---|---|
| Protocols | IGP underlay plus BGP overlay | BGP for underlay and overlay |
| Overlay topology | Route-reflected iBGP | Leaf-to-spine eBGP propagation |
| VTEP next hop | Naturally preserved through route reflection | Must be explicitly preserved through transit spines |
| Route Targets | Auto-derived RTs are simple with one ASN | Manual RTs or domain-wide rewrite commonly required |
| Failure domains | Shared IGP and ASN domains | Natural per-session and per-rack AS boundaries |
| Policy | Centralized and relatively minimal | Fine-grained at every eBGP boundary |
| Troubleshooting | Separate IGP and EVPN views | One protocol, but more address families and policy knobs |
| Scaling | Well suited to many enterprise fabrics | Attractive for very large, automation-driven Clos fabrics |
| Main risk | RR/IGP design or hidden dependency | Silent drops caused by next-hop, RT, or AS-path policy |

#### BGP unnumbered

BGP unnumbered typically uses IPv6 link-local next hops on point-to-point fabric links while advertising IPv4 NLRI. This removes most per-link IPv4 addressing and lets automation derive neighbors from cabling. The current standards reference is [RFC 8950](https://www.rfc-editor.org/rfc/rfc8950.html), which obsoletes RFC 5549. Platform support, interface discovery, extended-next-hop capability negotiation, and troubleshooting tooling must all be validated before adoption.

#### Practical selection guidance

For an enterprise fabric where operational familiarity, vendor reference designs, and minimal overlay policy matter most, an IGP underlay with iBGP EVPN is often the lower-friction choice. It is particularly attractive when one team already operates OSPF or IS-IS well and the fabric fits comfortably within the vendor's validated scale.

eBGP everywhere becomes compelling when the organization wants a uniform BGP operating model, explicit per-rack failure domains, strong automation, and a scale at which IGP and route-reflector design become material concerns. Its additional commands are manageable when generated and validated from a source of truth; they are risky when configured manually and inconsistently.

Convergence is not inherently won by either model. It depends on link-failure detection, BFD or equivalent mechanisms where appropriate, route propagation, ECMP programming, hardware behavior, and the number of affected prefixes. Measure failure and restoration under realistic load.

A hybrid eBGP underlay with an iBGP EVPN overlay is valid and is deployed in some designs. It can preserve the familiar iBGP EVPN control plane while using eBGP underlay failure domains, but it also retains two BGP session types and makes overlay session reachability dependent on the eBGP underlay. Choose it only when those trade-offs are intentional and supported by a reference design—not merely as an accidental midpoint.

### 4.2 BUM Handling

BUM handling spans three parts of the architecture and is mostly orthogonal to whether the EVPN overlay uses iBGP or eBGP:

- **EVPN control plane:** Type 3 Inclusive Multicast Ethernet Tag (IMET) routes advertise VTEP participation and replication-tunnel information.
- **Ingress VTEP:** decides whether to make multiple unicast copies or send one packet into an underlay multicast tree.
- **Underlay:** transports the resulting unicast VXLAN packets or performs multicast-tree replication.

The two primary mechanisms are **ingress replication (IR)** and **multicast underlay**. In the standards model, RFC 8365 uses the PMSI Tunnel attribute on the Type 3 route to identify the multicast-tunnel type; defined choices include ingress replication, PIM-SM, PIM-SSM, and BIDIR-PIM. The BGP Encapsulation Extended Community identifies VXLAN as the tunnel encapsulation. Implementations can differ; notably, Cumulus Linux does not advertise a Type 3 route for an L2 VNI whose BUM mode is PIM-SM. [RFC 8365, Section 9](https://www.rfc-editor.org/rfc/rfc8365.html#section-9) [NVIDIA EVPN deployment scenarios](https://docs.nvidia.com/networking-ethernet-software/guides/EVPN-Network-Reference/EVPN-Deployment-Scenarios/#multicast-replication)

![BUM ingress replication compared with multicast-underlay replication](/posts/vxlan-evpn-architecture/bum_ingress_replication_vs_multicast.svg)

#### Ingress replication

With IR, every VTEP advertising membership in an L2 VNI becomes a candidate remote destination. The ingress VTEP builds a head-end replication list from received Type 3 routes and sends one unicast VXLAN copy to each eligible remote VTEP.

For a VNI active on 50 VTEPs, a BUM frame arriving on one VTEP can produce up to 49 outgoing VXLAN copies. Replication consumes ingress-leaf bandwidth and hardware replication resources, but the spines maintain only ordinary unicast forwarding state.

IR advantages are:

- No PIM, RP, or multicast group state in the underlay.
- The same IP unicast reachability used by known-unicast VXLAN also transports BUM.
- A natural fit for BGP-only and automation-first fabrics.
- Straightforward failure behavior: a withdrawn Type 3 route removes a destination from the replication list.

IR costs are:

- Replication grows roughly with the number of remote VTEPs in each VNI.
- A BUM-heavy workload can consume substantial bandwidth on the ingress leaf's uplinks.
- Overlay multicast is replicated as unicast copies unless more specialized multicast features are deployed.
- Replication-list and NVE-peer scale must be included in the hardware capacity budget.

#### Multicast underlay

With multicast replication, an L2 VNI maps to an underlay multicast group. The ingress VTEP sends one VXLAN packet to that group, and the PIM tree replicates it only where the underlay branches. The VTEPs participating in the VNI join the corresponding tree.

Multicast-underlay advantages are:

- One copy leaves the ingress VTEP regardless of the number of remote VTEPs.
- Replication occurs at efficient branch points in the fabric.
- It scales better for VNIs with many VTEPs and significant BUM or overlay-multicast volume.

Its costs are additional PIM operations, RP design, multicast RPF dependencies, and multicast state on fabric nodes. Troubleshooting now requires correlating the EVPN Type 3 route, VNI-to-group mapping, PIM neighbor state, RPF result, and multicast forwarding tree.

#### How replication maps to the BGP design choices

An **IGP underlay with iBGP EVPN** commonly uses multicast replication in classic Nexus validated designs. PIM-SM runs on the routed fabric alongside OSPF or IS-IS; the IGP supplies unicast RPF reachability toward VTEP and RP loopbacks; and redundant spines commonly provide Anycast RP.

An **eBGP-everywhere** fabric commonly uses ingress replication. This preserves the operational goal of using BGP and unicast forwarding without adding PIM state to the spines. That is a convention, not a protocol requirement: PIM can use routes learned through eBGP for RPF. If multicast is added to a BGP-only underlay, validate the platform's ECMP RPF behavior, PIM convergence, and vendor-supported topology.

IR itself is independent of the underlay routing protocol. Once the remote Type 3 routes and VTEP next hops are valid, the ingress VTEP sends the same unicast copies whether those loopbacks are reachable through OSPF, IS-IS, iBGP, or eBGP.

#### Configuring a multicast underlay on NX-OS

The following pattern uses IPv4 PIM Sparse Mode with redundant spine Anycast RPs. It deliberately excludes ingress replication for the listed VNIs.

1. Enable PIM sparse mode on every routed fabric link.
2. Enable PIM on the relevant loopbacks, including the NVE source loopback where required by the platform design.
3. Place redundant RPs in the spine tier.
4. Advertise the unique RP and shared Anycast-RP loopbacks through the underlay.
5. Configure the same RP group scope on all participating devices.
6. Map every L2 VNI to its intended multicast group under `nve1`.

NX-OS supports native PIM Anycast RP, which avoids a separate MSDP mesh inside this fabric pattern. Traditional Anycast RP with MSDP is another design, but the two approaches should not be combined accidentally.

Spine example:

```text
feature pim

interface loopback1
  description SHARED-ANYCAST-RP
  ip address 10.0.100.1/32
  ip pim sparse-mode

interface loopback2
  description UNIQUE-RP-ID
  ip address 10.0.1.1/32
  ip pim sparse-mode

ip pim rp-address 10.0.100.1 group-list 239.1.0.0/16
ip pim anycast-rp 10.0.100.1 10.0.1.1
ip pim anycast-rp 10.0.100.1 10.0.1.2
```

Both spines receive the complete Anycast-RP peer list. `10.0.100.1` is configured identically on each RP, while `10.0.1.1` and `10.0.1.2` are unique addresses identifying the individual spines.

Leaf example:

```text
feature pim

ip pim rp-address 10.0.100.1 group-list 239.1.0.0/16

interface Ethernet1/1
  description TO-SPINE-1
  no switchport
  ip pim sparse-mode

interface Ethernet1/2
  description TO-SPINE-2
  no switchport
  ip pim sparse-mode

interface loopback0
  description NVE-SOURCE
  ip address 10.255.0.11/32
  ip pim sparse-mode

interface nve1
  no shutdown
  source-interface loopback0
  host-reachability protocol bgp

  member vni 30001
    mcast-group 239.1.1.1

  member vni 30002
    mcast-group 239.1.1.2
```

Current NX-OS also supports global L2 multicast-group configuration with per-VNI overrides on supported releases. Cisco explicitly documents that a multicast group can be configured per L2 VNI and that ingress replication is the alternative. [Cisco Nexus 9000 VXLAN Configuration Guide, Release 10.6(x)](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/106x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-106x.pdf)

#### Multicast group-allocation strategy

| Strategy | Benefit | Cost |
|---|---|---|
| One group per VNI | Best receiver and failure isolation; VTEPs receive BUM only for that VNI | Highest multicast-group and tree-state consumption |
| One group per tenant or VNI block | Balances group scale with useful scoping | VTEPs may receive traffic for other VNIs sharing the group and discard it after VXLAN lookup |
| One group shared broadly | Minimizes multicast group count | Poor flood scoping and potentially large amounts of unwanted BUM delivery |

The correct allocation depends on the maximum supported multicast groups, number of VTEPs per VNI, expected BUM rate, and hardware replication architecture. Document the mapping rather than generating it implicitly with an undocumented formula.

#### PIM ASM versus BIDIR-PIM

PIM ASM Sparse Mode with Anycast RP is widely deployed and broadly understood. Each active VTEP source can create source-specific state and may transition toward a shortest-path tree.

BIDIR-PIM maintains one bidirectional shared tree per multicast group and avoids per-source `(S,G)` state. This can reduce state and SPT-switching churn when every VTEP can be both a source and receiver. On supported Nexus platforms, redundant BIDIR designs commonly use phantom RP rather than the ASM Anycast-RP model. Platform support and group-range configuration must be verified before choosing it. [Cisco Nexus VXLAN underlay design guide](https://www.cisco.com/c/en/us/td/docs/dcn/whitepapers/cisco-vxlan-bgp-evpn-design-and-implementation-guide.html)

#### How Anycast RP synchronizes source information

Anycast RP gives several physical RPs the same logical RP address. Unicast routing sends a source Designated Router's PIM Register to its nearest RP, while a receiver's `(*,G)` Join can reach a different physical RP. Sharing the address therefore provides reachability and fast failover, but it does **not** by itself tell every RP which sources are active. The RP set needs a source-synchronization mechanism.

Two mechanisms are commonly encountered:

- **Anycast RP with MSDP:** the RP that learns a source originates an MSDP Source-Active (SA) message. The other RPs learn the `(S,G)` and can join toward the source when they have interested receivers. This is the traditional Anycast-RP design described by RFC 3446.
- **PIM Anycast-RP (RFC 4610):** the RP receiving a Register from a source DR copies that Register to the unique addresses of the other RP-set members. Each receiving RP creates `(S,G)` state and can deliver traffic down its own shared tree. A Register received from another configured RP is not copied again, which prevents a replication loop.

RFC 4610 removes the internal MSDP dependency, but it requires native support on every active RP and a consistently configured, deliberately small RP set. MSDP has more protocol machinery and peer state, yet remains necessary on platforms that do not implement RFC 4610 or when source discovery must cross PIM domains. Do not configure both methods between the same internal Anycast-RP members unless the vendor explicitly documents the interaction. [RFC 4610](https://www.rfc-editor.org/rfc/rfc4610.html) [RFC 3446](https://www.rfc-editor.org/rfc/rfc3446.html)

#### Why ACI IPN commonly uses BIDIR while generic EVPN often uses ASM

The choice follows multicast-state economics rather than a difference in VXLAN encapsulation.

An ACI Inter-Pod Network can carry many infrastructure multicast groups, with many leaf or spine endpoints acting as both sources and receivers. Under ASM, that pattern can create substantial `(S,G)` state, Register processing, and shortest-path-tree transitions. BIDIR-PIM keeps traffic on a bidirectional shared tree and maintains `(*,G)` rather than per-source state in the core. A phantom RP provides a stable RPF vector without requiring the RP address to terminate on one physical router. This makes BIDIR attractive for a dense, many-to-many infrastructure workload.

A general-purpose EVPN fabric often has fewer multicast groups, fewer active VTEP sources per group, or enough multicast-state capacity that ASM's per-source state is acceptable. ASM with Anycast RP is also supported across a broader range of switching platforms and appears in more vendor reference designs. Consequently, ASM is frequently the conservative interoperability choice, while BIDIR is selected when its state reduction is material and every device in the path supports it.

This is a design tendency, not a protocol rule. Estimate state before selecting the mode:

```text
ASM source state       approximately active_sources_per_group x groups
BIDIR shared-tree state approximately groups
```

The estimate is intentionally simplified; actual hardware consumption also depends on tree branching, outgoing-interface lists, VRFs, and platform implementation. In particular, do not assume that a Cumulus/FRR-based fabric supports BIDIR merely because RFC 8365 defines a BIDIR-PIM tunnel type—verify the exact software and ASIC release.

#### Cumulus Linux and NVUE specifics

Cumulus Linux uses PIM-SM plus an MSDP full mesh for redundant Anycast RPs. Current NVIDIA documentation states that Cumulus supports one MSDP mesh group, requires all RPs in the domain to be members, and does not forward a received SA message onward. The resulting full mesh is therefore mandatory rather than optional. [NVIDIA Cumulus Linux PIM documentation](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux/Layer-3/Protocol-Independent-Multicast-PIM/#msdp)

The following Cumulus Linux 5.x NVUE sketch uses two RPs. Apply the RP mapping on every PIM router; configure the MSDP mesh only on the RPs. Interface names and some NVUE paths can vary between 5.x releases, so confirm them with `nv list-commands` and the documentation for the installed release.

RP loopbacks and group mapping:

```text
# On RP1; use a different unique /32 on RP2.
nv set interface lo ip address 10.10.10.101/32
nv set interface lo ip address 10.100.100.100/32

# On every PIM router.
nv set vrf default router pim address-family ipv4 rp 10.100.100.100 group-range 239.1.0.0/16

# Enable PIM on every routed fabric interface that participates in the tree.
nv set interface swp51 router pim
nv set interface swp52 router pim
```

MSDP mesh on RP1 and RP2:

```text
# RP1
nv set vrf default router pim msdp-mesh-group FABRIC-RPS member-address 10.10.10.102
nv set vrf default router pim msdp-mesh-group FABRIC-RPS source-address 10.10.10.101

# RP2
nv set vrf default router pim msdp-mesh-group FABRIC-RPS member-address 10.10.10.101
nv set vrf default router pim msdp-mesh-group FABRIC-RPS source-address 10.10.10.102

nv config apply
```

For a global L2VNI-to-group mapping on each VTEP:

```text
nv set nve vxlan flooding multicast-group 239.1.1.1
nv config apply
```

For per-VNI mappings, NVIDIA documents `vxlan-mcastgrp` in `/etc/network/interfaces`:

```text
auto vni30001
iface vni30001
    bridge-access 101
    vxlan-id 30001
    vxlan-mcastgrp 239.1.1.1
    bridge-learning off
    bridge-arp-nd-suppress on

auto vni30002
iface vni30002
    bridge-access 102
    vxlan-id 30002
    vxlan-mcastgrp 239.1.1.2
    bridge-learning off
    bridge-arp-nd-suppress on
```

After changing `/etc/network/interfaces`, use `ifreload -a` during an appropriate change window. NVIDIA notes that one group per L2 VNI gives the best underlay bandwidth isolation, while sharing groups reduces multicast state at the expense of sending some VTEPs traffic for VNIs they do not host. [NVIDIA EVPN BUM with PIM-SM](https://docs.nvidia.com/networking-ethernet-software/cumulus-linux-58/Network-Virtualization/Ethernet-Virtual-Private-Network-EVPN/EVPN-PIM/)

There are three Cumulus-specific caveats worth making explicit:

1. **No Type 3 advertisement in PIM-SM mode:** validate the multicast tree and VNI/group mapping directly; the absence of a Type 3 route for that VNI is expected Cumulus behavior, not automatically a fault.
2. **Unnumbered BGP:** advertise the Anycast-RP address for reachability, but do not use it to form unicast or multicast peerings. Use each RP's unique address as the MSDP source and, where required, the PIM hello source.
3. **RP placement in an eBGP Clos:** NVIDIA currently recommends not using a spine as RP in this topology. Treat the earlier spine-RP NX-OS example as platform-specific, and follow the Cumulus reference topology when deploying Cumulus with eBGP.

Useful Cumulus checks include:

```text
nv show vrf default router pim
nv show interface swp51 router pim
sudo vtysh -c 'show ip pim neighbor'
sudo vtysh -c 'show ip pim rp-info'
sudo vtysh -c 'show ip msdp peer'
sudo vtysh -c 'show ip msdp sa'
sudo vtysh -c 'show ip mroute'
ip -d link show type vxlan
```

Verify both the wildcard `(*,G)` entry and active `(S,G)` entries, their incoming-interface RPF choice, and their outgoing-interface lists. Then capture on a fabric link and confirm that the outer destination is the configured multicast group and the inner frame belongs to the expected VNI.

#### Consistency and failure modes

Keep one replication method and a consistent group mapping for a given VNI unless the platform explicitly documents mixed-mode interoperability. A Type 3/PMSI mismatch, inconsistent group, missing PIM join, or failed RPF check can silently blackhole only BUM traffic. The resulting symptom is deceptive: known-unicast traffic may work while ARP, unknown destinations, or overlay multicast fail intermittently.

Validate multicast BUM in this order:

1. Confirm the L2 VNI is operational on every intended VTEP.
2. Where the implementation uses EVPN IMET signaling for this mode, confirm every VTEP advertises and imports the correct Type 3 route. On Cumulus PIM-SM, its absence is expected.
3. Where present, confirm the PMSI tunnel type; in all cases, confirm that the VNI-to-group mapping agrees on every VTEP.
4. Confirm PIM neighbors on all routed fabric links.
5. Confirm RP reachability and RPF toward each VTEP source.
6. Confirm the expected `(*,G)` and `(S,G)` or BIDIR state.
7. Capture the outer packet and verify source VTEP, destination group, UDP/4789, and VNI.

#### BUM multicast is not Tenant Routed Multicast

The multicast groups in this section carry Layer 2 BUM for an L2 VNI. **Tenant Routed Multicast (TRM)** distributes routed customer multicast between subnets and VRFs, using L3 VNIs and additional MVPN-style control-plane procedures. Keep BUM and TRM group ranges, capacity budgets, configuration, and troubleshooting workflows separate.

## 5. VXLAN data-plane forwarding

Forwarding depends on endpoint location and whether the destination is known.

### 5.1 Local switching

When source and destination MAC addresses are attached to the same leaf and bridge domain, the frame is switched locally. No VXLAN header is added.

### 5.2 Known remote unicast

When the destination MAC is known behind another VTEP:

1. The ingress leaf learns or receives the mapping between the destination MAC and remote VTEP.
2. It maps the local bridge domain to the correct VNI.
3. It encapsulates the original frame in VXLAN/UDP/IP.
4. The underlay routes the outer IP packet to the remote VTEP.
5. The egress VTEP removes the outer headers and sends a normal Ethernet frame to the destination.

![VXLAN known-unicast forwarding packet walk](/posts/vxlan-evpn-architecture/known-unicast-forwarding.svg)

### 5.3 BUM traffic

**BUM** means broadcast, unknown Layer 2 unicast, and multicast traffic. One ingress frame may need to reach multiple VTEPs. Two common replication models are:

- **Underlay multicast:** a VNI is associated with an IP multicast group. The ingress VTEP sends one copy, and the multicast-enabled underlay builds the replication tree.
- **Ingress replication:** the ingress VTEP sends one unicast copy to every interested remote VTEP. This avoids multicast in the underlay but consumes more bandwidth and replication resources as the VTEP count grows.

Traditional flood-and-learn VXLAN is data driven. VTEPs discover remote source MACs from received VXLAN frames. This works, but it makes BUM handling part of endpoint discovery and has several limits:

- Flooding wastes bandwidth and creates large MAC tables.
- Endpoint mobility is harder to converge cleanly.
- A centralized gateway creates traffic hairpinning.
- There is no native control-plane validation of endpoint ownership.
- Troubleshooting depends heavily on observed data-plane behavior.

## 6. What EVPN adds

EVPN uses the MP-BGP L2VPN EVPN address family as a control plane for the VXLAN overlay. Leaf VTEPs advertise MAC addresses, IP bindings, IP prefixes, and tunnel membership. Remote VTEPs can install forwarding state before data arrives.

Key benefits are:

- Control-plane distribution of Layer 2 MAC and Layer 3 IP reachability.
- Reduced flooding and localized learning.
- ARP/ND suppression.
- Explicit endpoint-mobility signaling.
- Distributed anycast gateways.
- A unicast alternative to multicast for BUM replication.
- Route policies, authentication, and scalable route reflection through BGP.

### 6.1 Route reflectors

In a small fabric, every leaf could peer with every other leaf. At scale, route reflectors reduce the overlay BGP session count. Leaves advertise EVPN routes to the route reflectors, which reflect them to the other VTEPs. The route reflector does not have to be in the VXLAN data path.

![EVPN host and subnet distribution through route reflectors](/posts/vxlan-evpn-architecture/evpn-route-reflection.svg)

### 6.2 Underlay versus overlay BGP

If BGP is used in both layers, keep their responsibilities clear:

- **Underlay address family:** advertises infrastructure links and VTEP loopbacks.
- **L2VPN EVPN address family:** advertises overlay reachability and membership.

The two control planes can use the same BGP process but carry different NLRI and policies. An EVPN route's BGP next hop normally resolves through the underlay.

### 6.3 EVPN address family and route installation

EVPN NLRI uses AFI 25 (L2VPN) and SAFI 70 (EVPN). BGP transports the NLRI plus ordinary path attributes and EVPN-specific extended communities. A received route is useful only when all of the following succeed:

1. The BGP path is accepted and selected.
2. Its Route Target matches a local MAC-VRF or IP-VRF import policy.
3. Its encapsulation is supported, normally VXLAN.
4. The VNI and route-type fields are valid for the local service.
5. The BGP next hop resolves through the underlay.
6. Any required overlay index, such as a gateway IP, MAC, or ESI for a Type 5 route, resolves recursively.
7. Hardware resources are available to program the MAC, neighbor, tunnel, and route entries.

This explains a common troubleshooting pattern: a route can appear in `show bgp l2vpn evpn` yet be absent from the MAC table or tenant VRF. BGP receipt is only one stage of installation.

## 7. EVPN building blocks: RD, RT, and route types

### 7.1 Route Distinguisher

An **RD** is an 8-byte value prepended to EVPN NLRI to make otherwise identical routes unique. It is not an import/export policy. Two tenants can use the same MAC or IP space and still originate distinct VPN routes because their RDs differ.

Common formats include ASN:number and IP-address:number. Automatic derivation often uses a router ID or VTEP-specific value so different VTEPs originate unique routes.

### 7.2 Route Target

A **Route Target (RT)** is an extended community used as policy. An exporting VRF or VNI attaches an RT; an importing VRF or VNI accepts routes carrying the matching RT.

This gives a useful mental model:

- RD answers: **How is this route made globally unique?**
- RT answers: **Which routing or bridging domains should import it?**

![RD and RT policy concepts in MP-BGP EVPN](/posts/vxlan-evpn-architecture/rd-rt-policy.svg)

### 7.3 Important EVPN route types

RFC 7432 defines EVPN route Types 1 through 4. RFC 9136 later defines the IP Prefix route, Type 5. VXLAN fabrics primarily rely on Types 2, 3, and 5; multihoming additionally uses Types 1 and 4.

| Type | Name | Main VXLAN EVPN purpose |
|---|---|---|
| 1 | Ethernet Auto-Discovery | Multihoming aliasing, mass withdrawal, and Ethernet-segment signaling |
| 2 | MAC/IP Advertisement | Advertises a host MAC and optionally its IP binding |
| 3 | Inclusive Multicast Ethernet Tag | Signals VTEP membership and builds BUM replication lists |
| 4 | Ethernet Segment | Discovers VTEPs attached to the same multihomed Ethernet segment and supports DF election |
| 5 | IP Prefix | Advertises IP prefixes independently of individual host MAC routes |

#### Type 2: MAC and MAC/IP reachability

A Type 2 route can carry only a MAC or a MAC plus IP address. The route normally includes the RD, Ethernet Segment Identifier when relevant, Ethernet tag, MAC length and address, IP length and address, MPLS-label fields repurposed to carry the VNI, and the advertising VTEP as BGP next hop.

When the IP is present, remote VTEPs can populate both forwarding and neighbor-suppression state. This is one reason control-plane learning reduces ARP flooding.

In a VXLAN encoding, fields named “MPLS Label” by the original EVPN specification carry a 24-bit VNI. Label1 normally identifies the MAC-VRF/L2 VNI. Symmetric IRB can use Label2 to identify the IP-VRF/L3 VNI. RFC 8365 defines the EVPN-to-overlay mapping; treating these fields as literal MPLS labels in a VXLAN packet walk is incorrect. [RFC 8365](https://www.rfc-editor.org/rfc/rfc8365.html)

#### Type 3: inclusive multicast membership

A Type 3 route tells other VTEPs that the originator participates in a VNI. With ingress replication, the received Type 3 next hops become the head-end replication list. With multicast replication, the route can convey the provider multicast service information associated with the VNI.

#### Type 5: IP prefix reachability

Type 5 routes carry IP prefixes for a tenant VRF. They are useful for external routes, summarized subnets, border-leaf advertisements, and prefix-based routing that does not need one Type 2 route per endpoint.

Two common models appear in deployments:

- A prefix is advertised with a recursive next hop or router MAC so the remote VTEP routes it through the L3 VNI.
- A subnet is represented through an IRB/SVI context, depending on platform and design.

Type 5 deliberately decouples an IP prefix from a host MAC. It can carry an **overlay index**—a gateway IP address, router MAC, or ESI—that the receiving NVE resolves recursively to an egress VTEP. If the required overlay index cannot be resolved, the prefix cannot be installed for forwarding even if the Type 5 BGP path itself is valid. [RFC 9136, Sections 2-3](https://www.rfc-editor.org/rfc/rfc9136.html#section-3)

## 8. EVPN multihoming in detail

EVPN multihoming connects one customer edge, server bond, switch, firewall, or downstream network to two or more VTEPs without relying on a single physical leaf. This is different from MAC mobility: multihoming makes a MAC legitimately reachable through multiple PEs on the **same Ethernet segment**, whereas mobility means the endpoint moved between different Ethernet segments. [RFC 7432, Section 15](https://www.rfc-editor.org/rfc/rfc7432.html#section-15)

### 8.1 Ethernet Segment Identifier

An **Ethernet Segment (ES)** is the set of links connecting a multihomed device or network to the participating VTEPs. A nonzero 10-byte **Ethernet Segment Identifier (ESI)** identifies it. All VTEPs attached to the same ES must derive or configure the same ESI, while unrelated segments must not collide.

Redundancy modes are:

- **All-active:** all participating VTEPs can forward known unicast traffic to and from the multihomed device. The device commonly sees an LACP bundle.
- **Single-active:** only one VTEP forwards user traffic for a given service at a time. This is appropriate when the attached device cannot safely receive active-active traffic.

Cisco fabrics may implement dual-homing with vPC, standards-based EVPN ESI multihoming, or platform-specific variants. These are not interchangeable designs: verify support for the selected Nexus model, line card, NX-OS release, and feature combination in the current configuration guide.

### 8.2 How route types 1 and 4 work together

**Type 4 - Ethernet Segment route** discovers which VTEPs participate in a nonzero ESI. It carries the ESI and originating router IP and supports DF election and ES-import policy.

**Type 1 - Ethernet Auto-Discovery route** has two important scopes:

- **Per-ES A-D route:** represents reachability through a VTEP to the whole Ethernet segment. Withdrawal can invalidate many dependent MAC paths at once, enabling fast or “mass” withdrawal after an access failure.
- **Per-EVI A-D route:** represents a VTEP's participation in a particular EVPN instance/VNI on that Ethernet segment. Remote VTEPs use it for aliasing and backup-path behavior.

Together, these routes let remote VTEPs build paths before every individual MAC has been learned through every multihoming VTEP.

### 8.3 Aliasing and known-unicast load balancing

Suppose a host MAC is learned and advertised by only one leaf in an all-active pair. A remote VTEP can still infer, from the Type 1 per-EVI routes, that the other leaf reaches the same ESI. **Aliasing** allows remote known-unicast traffic to use the full set of eligible VTEPs instead of being pinned to the leaf that originated the Type 2 route.

The remote VTEP must combine the Type 2 MAC route with the Type 1 Ethernet A-D state. Losing an A-D route removes that VTEP from the eligible next-hop set without waiting for every MAC route to be withdrawn independently.

### 8.4 Designated Forwarder election

For BUM traffic sent **toward** a multihomed Ethernet segment, only the Designated Forwarder should deliver a copy for the relevant Ethernet tag/service. Otherwise, parallel VTEPs could send duplicates to the attached device. RFC 7432 defines DF responsibility at the granularity of an Ethernet segment and Ethernet tag; later DF-election extensions can provide better distribution and convergence, but support is implementation-specific. [RFC 7432, Section 8.5](https://www.rfc-editor.org/rfc/rfc7432.html#section-8.5)

DF election does not mean all-active known unicast becomes single-active. It primarily controls multi-destination delivery and other explicitly defined actions.

### 8.5 Split horizon

Traffic received from an Ethernet segment must not be sent back to that same segment through another VTEP. EVPN split-horizon signaling identifies the originating ES so the receiving PE can suppress this loop. In an MPLS EVPN network this uses an ESI label; VXLAN implementations map equivalent split-horizon semantics to the overlay behavior described by RFC 8365 and platform mechanisms.

### 8.6 Failure sequences

An access-link or attached-device failure should produce this sequence:

1. The local VTEP removes its Ethernet-segment reachability.
2. The corresponding Type 1 route is withdrawn.
3. Remote VTEPs remove that VTEP from the aliasing next-hop set.
4. Remaining all-active VTEPs continue forwarding, or a new single-active/DF role is selected.
5. Individual Type 2 routes converge as necessary without being the only fast-failure signal.

Validate failure behavior independently for local-to-remote known unicast, remote-to-local known unicast, BUM traffic, and routed traffic through the anycast gateway.

## 9. Endpoint learning, mobility, and ARP suppression

### 9.1 Local and remote learning

A leaf learns a locally attached host from a data-plane source MAC, ARP/ND, or another authorized local mechanism. It then advertises a Type 2 route. Remote VTEPs install the MAC-to-VTEP mapping and, when present, the IP-to-MAC binding.

### 9.2 Host mobility

When a host moves from VTEP-1 to VTEP-3, VTEP-3 advertises the host with a higher MAC mobility sequence number. Other VTEPs prefer the newer advertisement and update their forwarding entry. The sequence provides an ordered control-plane signal and avoids waiting for an old MAC-table entry to age out.

The first advertisement does not need a MAC Mobility extended community; its effective sequence is zero. A VTEP that learns the same MAC locally on a different ESI advertises a sequence one greater than the highest received sequence. A receiver prefers the higher sequence; equal sequences on different ESIs use the lower advertising PE IP as the RFC-defined tie-breaker. Static/sticky MACs use the static flag and must not be treated as ordinary moves. [RFC 7432, Section 15](https://www.rfc-editor.org/rfc/rfc7432.html#section-15)

Duplicate or rapidly oscillating advertisements should be treated as an operational warning. RFC 7432's default duplicate-MAC detection example is five moves within 180 seconds, after which the PE alerts the operator and suppresses further route processing for that MAC until corrective action. Vendor defaults and recovery behavior may differ, so verify the live platform rather than relying on these numbers as universal configuration.

### 9.3 ARP suppression

With ARP suppression, an ingress VTEP uses its EVPN-learned IP/MAC database to answer an attached host's ARP request locally. The request does not need to be flooded across the VNI.

![ARP suppression using EVPN-learned IP and MAC state](/posts/vxlan-evpn-architecture/arp-suppression.svg)

If the lookup misses, the VTEP still replicates the request to the appropriate remote VTEPs. Suppression reduces known-neighbor broadcasts; it does not eliminate the need for a valid BUM mechanism.

![ARP lookup miss and controlled replication](/posts/vxlan-evpn-architecture/arp-miss.svg)

IPv6 uses the analogous Neighbor Discovery suppression behavior when supported.

ARP and ND suppression are hardware- and release-dependent features. On Nexus platforms, the SVI state, anycast-gateway configuration, TCAM allocation, and consistency across VTEPs can affect support. Current NX-OS documentation also lists combinations where ND suppression is unavailable or restricted, including some Multi-Site, vPC, IRB, and firewall scenarios. Always check the exact release and platform matrix before enabling suppression fabric-wide. [Cisco Nexus 9000 VXLAN Configuration Guide, Release 10.5(x)](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x.pdf)

## 10. Distributed anycast gateway and IRB

A centralized gateway forces traffic to cross the overlay to an aggregation device before it can be routed, then potentially cross back. This adds hops, concentrates state, and creates a scaling and convergence bottleneck.

With a **distributed anycast gateway**, every participating leaf presents the same default-gateway IP and virtual MAC for a subnet. A workload therefore finds its gateway locally wherever it moves. HSRP or VRRP is not required between all leaves for this function.

### 10.1 Same-subnet forwarding

Hosts in the same subnet and L2 VNI are bridged. If they are attached to different leaves, the ingress VTEP encapsulates the original Ethernet frame with the L2 VNI.

### 10.2 Inter-subnet forwarding

Hosts in different subnets require routing. VXLAN EVPN supports two IRB models.

![Asymmetric and symmetric VXLAN EVPN IRB comparison](/posts/vxlan-evpn-architecture/irb-comparison.svg)

#### Asymmetric IRB

The ingress VTEP performs the Layer 3 lookup, then sends the packet in the destination host's L2 VNI. The egress VTEP only bridges the final frame. Every VTEP that might route between tenant subnets must instantiate all relevant L2 VNIs, even with no local host in those segments.

This is simple to visualize but wastes forwarding resources and scales poorly as the number of tenant segments grows.

More precisely, RFC 9135 describes three ingress lookups for asymmetric IRB: destination-MAC lookup to the IRB interface, IP-VRF lookup, then destination-MAC lookup in the destination bridge table. The egress PE performs one MAC lookup. The ingress VTEP must therefore know the remote host's IP-to-MAC binding and must instantiate the destination subnet's bridge table and IRB interface even when that subnet has no local endpoint. [RFC 9135, Section 4](https://www.rfc-editor.org/rfc/rfc9135.html#section-4)

![Asymmetric IRB packet walk: the ingress VTEP routes into the destination L2 VNI and the egress VTEP only bridges](/posts/vxlan-evpn-architecture/asymmetric-irb-walk.svg)

#### Symmetric IRB

Both ingress and egress VTEPs perform routing. A tenant VRF has an L3 VNI:

1. The ingress VTEP receives a frame addressed to the anycast gateway MAC.
2. It performs a tenant-VRF lookup for the destination IP.
3. It rewrites the inner Layer 2 header for routed overlay forwarding, using the remote VTEP's router MAC as needed.
4. It encapsulates the packet with the tenant L3 VNI.
5. The egress VTEP decapsulates it and performs the second VRF lookup.
6. It resolves the destination host and sends the frame through the local L2 VNI/bridge domain.

![Symmetric IRB packet walk using an L3 VNI](/posts/vxlan-evpn-architecture/symmetric-irb-walk.svg)

The egress leaf needs only its locally used L2 VNIs plus the tenant L3 VNI. This is the more scalable model and is the design emphasized by the course.

In standards terminology, symmetric IRB forwards between the ingress and egress **IP-VRFs**. With Ethernet NVO encapsulation such as VXLAN, the inner source and destination MAC addresses are router MACs, not the final destination host MAC. This is why the ingress leaf does not need the remote host's ARP entry merely to carry routed traffic across the L3 VNI. The egress leaf resolves the local host after its IP-VRF lookup. [RFC 9135, Section 4](https://www.rfc-editor.org/rfc/rfc9135.html#section-4)

One subtle operational difference is TTL/hop-limit handling. RFC 9135 specifies a decrement at both ingress and egress routing PEs for symmetric IRB, but only at the ingress PE for asymmetric IRB. Packet captures and traceroute interpretation should account for the two routed overlay stages.

## 11. Head-end replication and control-plane suppression

The EVPN control plane minimizes flood-and-learn behavior but does not make every frame unicast. Unknown destinations and genuine broadcasts still require replication.

In head-end replication:

1. Each VTEP advertises VNI membership, commonly using a Type 3 route.
2. Other VTEPs construct an ingress-replication list for that VNI.
3. The ingress VTEP makes one VXLAN unicast copy per remote member.
4. Each receiver decapsulates and floods only to the appropriate local interfaces.

![EVPN head-end replication for BUM traffic](/posts/vxlan-evpn-architecture/head-end-replication.svg)

ARP suppression, Type 2 host learning, and Type 3 membership work together: known endpoint resolution remains local or unicast, while unavoidable multi-destination traffic follows a controlled replication list.

## 12. Design choices: pod, multi-pod, fabric, and site

The course distinguishes designs that are often casually grouped together.

### 12.1 Pod

A pod is a repeated leaf-spine building block. A fabric can grow by adding leaves until spine port capacity or scale limits are reached. Adding another group of spines and leaves creates another pod.

### 12.2 Multi-Pod

In a multi-pod design, pods are connected—often through a super-spine layer—but still form one logical fabric:

- One end-to-end overlay domain.
- One end-to-end EVPN control-plane domain.
- One extended underlay reachability domain.
- One BUM replication domain.
- One VNI administrative domain.

![Multi-pod architecture with a super-spine layer](/posts/vxlan-evpn-architecture/multi-pod.svg)

This gives simple end-to-end connectivity, but failure and operational domains grow with the fabric. A route or BUM event can propagate throughout all pods. VTEP-to-VTEP underlay reachability must also extend end to end.

### 12.3 Multi-fabric and Multi-Site

A multi-fabric architecture creates separate fabrics with isolated underlays and overlay control-plane domains. Connectivity between them is explicit and controlled. VXLAN Multi-Site is Cisco's architecture for this model, using border gateways to interconnect independent sites.

![Comparison of multi-pod and multi-fabric designs](/posts/vxlan-evpn-architecture/multipod-vs-multisite.svg)

Choose multi-pod when the operational simplicity of one domain outweighs the larger blast radius. Choose Multi-Site when isolation, independent change control, and selective inter-site extension are more important.

## 13. VXLAN Multi-Site architecture

Multi-Site preserves independent intra-site VXLAN EVPN fabrics while connecting selected Layer 2 and Layer 3 services across a data-center interconnect (DCI).

Unlike VXLAN and EVPN themselves, **VXLAN EVPN Multi-Site is a Cisco architecture with platform-specific control-plane and forwarding behavior**. Treat its PIP/VIP advertisements, route re-origination, tracking, and supported feature combinations as NX-OS implementation details, not generic RFC 7432 behavior.

### 13.1 Border gateway roles

A **Border Gateway (BGW)** is the key component. It participates in the local site's EVPN fabric and in the inter-site control and data planes. Its responsibilities include:

- Re-originating selected EVPN routes between site-local and inter-site domains.
- Terminating and originating inter-site VXLAN tunnels.
- Controlling which L2 VNIs and L3 VNIs extend across the DCI.
- Preventing loops and unnecessary BUM propagation.
- Preserving site isolation while providing reachability.

![VXLAN Multi-Site border-gateway architecture](/posts/vxlan-evpn-architecture/multisite-architecture.svg)

Each site keeps its local VTEP addressing, route-reflection design, and underlay. Remote sites do not need direct underlay routes to every internal VTEP; they reach the remote site's BGW function.

### 13.2 Underlay isolation

The DCI carries reachability between border gateways, not a merger of every site's internal underlay. This limits failure propagation and keeps internal VTEP prefixes private to a site.

![Multi-Site underlay isolation and BGW VTEP identities](/posts/vxlan-evpn-architecture/multisite-underlay-isolation.svg)

### 13.3 PIP and Multi-Site VIP

A BGW uses more than one tunnel identity:

- The **PIP** (Primary/Physical IP) identifies an individual BGW and is useful for traffic that must target that device specifically.
- The **Multi-Site VIP** is shared by the site's BGWs and represents the anycast border-gateway function to remote sites.

Remote BGWs can therefore send ordinary inter-site traffic to a site-level anycast VTEP while retaining individual reachability when a function requires the PIP.

The PIP/VIP choice is feature-dependent. Some designs advertise PIP reachability for individual next-hop selection, external connectivity, CloudSec, or specialized traffic engineering. Commands such as `advertise-pip`, `fabric-advertise-pip`, and `dci-advertise-pip` have release-, topology-, and underlay-specific restrictions. Do not infer the correct behavior from the command names alone.

### 13.4 Anycast BGW and designated forwarder

A site normally deploys multiple BGWs for redundancy. They share the Multi-Site VIP but retain individual PIPs. For multi-destination traffic, a designated-forwarder election ensures only the correct BGW forwards a given copy, preventing duplicate delivery.

![Anycast border gateways, VIP/PIP, and designated-forwarder behavior](/posts/vxlan-evpn-architecture/anycast-bgw.svg)

### 13.5 BUM replication modes

Multi-Site can use different replication mechanisms inside and between sites. For example, a site may use multicast internally while the DCI uses ingress replication. The BGW translates the replication behavior at the boundary.

![Multi-Site BUM replication with multicast in the underlay](/posts/vxlan-evpn-architecture/multisite-bum-multicast.svg)

![Mixed replication: intra-site multicast and inter-site ingress replication](/posts/vxlan-evpn-architecture/multisite-bum-mixed.svg)

### 13.6 Selective advertisement

Only VNIs configured and permitted on the BGW should cross the DCI. Selective advertisement improves scale and security:

- A local-only tenant does not consume remote-site state.
- A stretched L2 segment can be permitted without exposing unrelated segments.
- A tenant L3 VNI can extend routed reachability while its access VLANs remain local.

The BGW is therefore a policy boundary, not merely a tunnel relay.

### 13.7 Multi-Site control-plane boundaries

A robust design separates three adjacency scopes:

- **Intra-site underlay:** provides reachability among local leaves, spines, route reflectors, and BGW fabric-facing identities.
- **Intra-site EVPN overlay:** distributes local endpoint and tenant routes between site VTEPs and local BGWs.
- **Inter-site underlay and EVPN overlay:** provides DCI reachability and exchanges only the EVPN routes selected for extension.

BGWs re-originate routes at the boundary so remote sites see the site-level BGW next hop rather than every internal leaf. This limits underlay state and creates a clean failure boundary. A design that merely extends the same route-reflector and VTEP domain across sites is closer to Multi-Pod than Multi-Site.

### 13.8 Tracking and restoration

`evpn multisite fabric-tracking` identifies links toward the local fabric; `evpn multisite dci-tracking` identifies links toward the inter-site network. Tracking allows a BGW to stop acting as a usable transit node when it loses one side of the path. `delay-restore time` delays restoration after recovery so control-plane and forwarding state can stabilize before the BGW attracts traffic.

The site ID must be identical on all BGWs in one site and different between sites. Current Cisco guidance also requires explicit planning for NVE source loopbacks and the BGW VIP, with underlay reachability for both where applicable. [Cisco, Configure VXLAN EVPN Multi-Site](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/configuring-multisite.pdf)

## 14. Multi-Site forwarding walks

### 14.1 Inter-site bridging

For two hosts in the same stretched L2 VNI but different sites:

1. The local leaf forwards the frame toward its site BGW using the intra-site VXLAN overlay.
2. The BGW terminates the intra-site tunnel.
3. It applies inter-site policy and re-encapsulates toward the remote site's Multi-Site VIP.
4. A remote BGW decapsulates and sends a new intra-site VXLAN packet toward the destination leaf.
5. The destination leaf decapsulates and bridges the original frame to the host.

This is not one end-to-end tunnel. The BGWs divide the journey into independently controlled tunnel domains.

### 14.2 Inter-site routing

For hosts in different subnets or tenants with permitted reachability, symmetric IRB remains the basic model. Tenant routes are advertised through the L3 VNI, and the BGWs re-originate allowed reachability. Routing occurs in the tenant VRF while each site retains its own underlay.

### 14.3 Inter-site BUM

A local BUM frame is replicated within the source site and, if the VNI is stretched, to the selected BGW. The BGW sends controlled copies to remote sites. Each remote site then performs its own local replication. Split-horizon and DF logic prevent a copy from returning to its origin or being duplicated by parallel BGWs.

## 15. Representative Cisco NX-OS configuration model

The source dedicates many slides to a two-site lab. The following is a consolidated translation of its workflow. It is a pattern, not a paste-ready build.

The snippets intentionally show the configuration hierarchy, not a complete production configuration. They omit platform-specific TCAM carving, route policies, authentication, BFD, maximum-path settings, multicast RP configuration, QoS, telemetry, and management-plane hardening. Choose one documented NX-OS release as the source of truth and lab-test the exact switch image and line card.

### 15.1 Enable features

```text
feature ospf
feature bgp
feature pim
feature interface-vlan
feature vn-segment-vlan-based
feature nv overlay
nv overlay evpn
```

Depending on the platform, additional commands enable fabric forwarding, NGOAM, or Multi-Site functions.

### 15.2 Underlay loopback and routed link

```text
interface loopback0
  description ROUTER-ID_AND_VTEP
  ip address 10.1.1.2/32
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode

interface Ethernet1/1
  description TO-SPINE
  no switchport
  ip address 10.11.12.2/24
  ip router ospf UNDERLAY area 0.0.0.0
  ip pim sparse-mode
  no shutdown
```

The source lab uses OSPF and PIM in the underlay. A modern deployment may instead use eBGP underlay and ingress replication.

A dedicated VTEP loopback is generally preferable to reusing the BGP router ID. It makes tunnel-source migration, route policy, troubleshooting, and vPC/ESI-specific PIP/VIP behavior easier to reason about. Advertise it as a host route and ensure every ECMP path supports the overlay MTU.

### 15.3 Overlay BGP

```text
router bgp 65001
  router-id 10.1.1.2
  neighbor 10.1.1.1
    remote-as 65001
    update-source loopback0
    address-family l2vpn evpn
      send-community
      send-community extended
      route-reflector-client
```

The `route-reflector-client` line belongs on a route reflector, not on an ordinary leaf. A leaf's equivalent neighbor stanza omits it.

### 15.4 VLAN-to-VNI and tenant VRF

```text
vlan 100
  vn-segment 10100

vlan 1111
  name TENANT1-L3VNI
  vn-segment 50111

vrf context Tenant-1
  vni 50111
  rd auto
  address-family ipv4 unicast
    route-target both auto
    route-target both auto evpn
```

### 15.5 Anycast gateway SVI

```text
fabric forwarding anycast-gateway-mac 0000.2222.3333

interface Vlan100
  no shutdown
  vrf member Tenant-1
  ip address 192.168.1.254/24
  fabric forwarding mode anycast-gateway
```

The anycast gateway MAC and gateway IP must be consistent on all leaves that offer the subnet.

### 15.6 NVE interface

```text
interface nve1
  no shutdown
  host-reachability protocol bgp
  source-interface loopback0

  member vni 10100
    ingress-replication protocol bgp

  member vni 50111 associate-vrf
```

With multicast replication, an L2 VNI membership can reference a multicast group instead of BGP ingress replication.

### 15.7 EVPN VNI policy

```text
evpn
  vni 10100 l2
    rd auto
    route-target import auto
    route-target export auto
```

Automatic RT derivation is convenient inside one ASN. Across multiple autonomous systems or during migration, explicit RTs may be required so every site derives and imports compatible values.

Remember that the RT controls route import, whereas the VNI controls data-plane service identification. Two VNIs do not become one broadcast domain merely because their numbers look related, and two VRFs do not exchange routes unless their import/export policy permits it.

### 15.8 Multi-Site BGW pattern

```text
evpn multisite border-gateway 11
  delay-restore time 30

interface loopback100
  description MULTISITE-VIP
  ip address 10.10.10.10/32 tag 1234

interface nve1
  multisite border-gateway interface loopback100
  member vni 10100
    multisite ingress-replication

interface Ethernet1/2
  description DCI
  no switchport
  medium p2p
  evpn multisite dci-tracking

interface Ethernet1/3
  description FABRIC
  no switchport
  evpn multisite fabric-tracking
```

The site ID must be unique per site and identical across redundant BGWs in that site. DCI-facing and fabric-facing links are tracked separately so Multi-Site can react correctly to isolation and partial failures.

The current Cisco Multi-Site guide requires planning loopback addresses and confirming their underlay advertisement before enabling the BGW function. It also documents feature-specific restrictions—for example, some PIP advertisement combinations differ with vPC and IPv6 underlays—so this abbreviated template must not be used as a capability matrix. [Cisco Multi-Site configuration guide](https://www.cisco.com/c/en/us/td/docs/dcn/nx-os/nexus9000/105x/configuration/vxlan/cisco-nexus-9000-series-nx-os-vxlan-configuration-guide-release-105x/configuring-multisite.pdf)

![Two-site lab topology used by the source course](/posts/vxlan-evpn-architecture/two-site-lab.svg)

## 16. Failure handling and verification

Multi-Site should be tested for at least these events:

- A leaf-to-spine fabric link fails.
- One BGW loses all fabric-facing links.
- One BGW loses its DCI-facing links.
- A BGW reloads and returns.
- An underlay route to a VTEP or VIP disappears.
- A host moves within a site or between sites.
- A route reflector becomes unavailable.

Fabric tracking and DCI tracking help a BGW decide whether it is still a valid transit point. Delay-restore timers prevent a recovering device from advertising reachability before its forwarding state is ready.

![Dual-BGW Multi-Site failure-analysis topology](/posts/vxlan-evpn-architecture/dual-bgw-failure.svg)

Useful verification commands include:

```text
show bgp l2vpn evpn summary
show bgp l2vpn evpn
show bgp l2vpn evpn route-type 2
show bgp l2vpn evpn route-type 3
show bgp l2vpn evpn route-type 5
show nve peers
show nve vni
show nve multisite fabric-links
show nve multisite dci-links
show mac address-table dynamic
show ip arp vrf <vrf-name>
show forwarding route vrf <vrf-name>
```

Interpret the evidence by control-plane object:

| Symptom | Route/state to inspect | Typical root causes |
|---|---|---|
| Remote host MAC missing | Type 2 MAC route, MAC-VRF import RT | Local host never learned, export RT mismatch, BGP path rejected, unresolved VTEP next hop |
| MAC exists but ARP suppression misses | Type 2 MAC/IP route, suppression cache | Type 2 carries no IP, silent host, SVI/suppression mismatch, TCAM/platform limitation |
| BUM reaches no remote VTEPs | Type 3 IMET and NVE replication list | VNI not active, RT mismatch, missing PMSI tunnel information, multicast RPF failure |
| Multihomed next hop is not load-balanced | Type 1 per-EVI and Type 2/ESI state | ESI mismatch, A-D route absent, single-active mode, aliasing unsupported or filtered |
| Duplicate BUM at multihomed device | Type 4, DF state, split-horizon state | Inconsistent ESI, DF disagreement, missing ES-import policy, transient convergence |
| Prefix is visible but not installed | Type 5 plus overlay-index recursion | Missing RT-2/RT-1 dependency, unresolved BGP next hop, wrong IP-VRF RT, unsupported model |
| Inter-site route stops at BGW | Re-originated EVPN route, PIP/VIP, tracking | VNI not extended, DCI policy, site-ID inconsistency, fabric/DCI tracking withdrawal |

A disciplined troubleshooting order is:

1. Confirm physical links, MTU, and routed underlay adjacency.
2. Confirm VTEP loopback and Multi-Site VIP reachability.
3. Confirm EVPN BGP sessions and correct address-family activation.
4. Check Type 3 membership before debugging BUM replication.
5. Check Type 2 MAC/IP routes for endpoint reachability.
6. Check Type 5 and VRF import policy for prefix reachability.
7. Verify VLAN-to-VNI, VRF-to-L3-VNI, RD, and RT mappings.
8. Inspect the NVE peer and local MAC/ARP tables.
9. Capture the packet and compare inner versus outer headers.

## 17. Multi-tenant connectivity with firewall insertion

VXLAN EVPN isolates tenant VRFs by default. Connecting tenants is a policy decision. The first multi-tenant design in the course inserts an ASAv firewall between tenant domains.

![Multi-tenant dual-site topology with centralized firewall insertion](/posts/vxlan-evpn-architecture/firewall-insertion.svg)

Two questions must be answered:

1. How does a border or service leaf pass traffic between tenant VRFs and the firewall?
2. How do remote leaves steer inter-tenant traffic toward that border/service leaf?

A common design uses dedicated transit VLANs or routed interfaces between the service leaf and firewall. Each tenant VRF sends selected routes or a default route to the firewall. The firewall owns inter-tenant policy and returns traffic into the appropriate tenant-facing interface.

Important details include:

- Keep each tenant in a distinct firewall zone or subinterface.
- Advertise only the routes needed to reach the service insertion point.
- Avoid importing tenant routes directly into one another when inspection is mandatory.
- Ensure the forward and return directions traverse the same stateful firewall context.
- Consider redundancy, ECMP support, and session synchronization for firewall pairs.

The design is centralized from the policy perspective, even though endpoints and gateways remain distributed.

## 18. Centralized route leaking and shared services

The final course section builds a third tenant VRF that acts as a shared Internet/services domain. It then leaks selected reachability between Tenant-1 and Tenant-3 at a border leaf.

![Multi-tenant topology for centralized route leaking and shared Internet](/posts/vxlan-evpn-architecture/central-route-leaking.svg)

### 18.1 Why centralize route leaking

If every leaf imports routes between tenant VRFs, policy is distributed throughout the fabric and becomes hard to audit. Centralized route leaking restricts the import/export logic to designated border leaves. Other leaves use EVPN to reach those border nodes.

The basic policy is:

- Tenant-3 learns or originates Internet/default reachability.
- Tenant-1 imports only the Tenant-3 routes it needs, often a default route.
- Tenant-3 imports the selected Tenant-1 prefixes required for the return path.
- Route maps and RT policies prevent accidental full-mesh tenant connectivity.

### 18.2 Shared Internet model

The source lab uses Tenant-3 as the shared Internet VRF. Tenant-1 reaches the Internet through Tenant-3 while remaining otherwise isolated. The border leaf performs controlled route leaking, and a WAN-facing interface or external peer supplies default reachability.

![Shared Internet through a Tenant-3 services VRF](/posts/vxlan-evpn-architecture/shared-internet-vrf.svg)

A conceptual NX-OS policy looks like this:

```text
route-map T1_IMPORT permit 10
  match tag 200

route-map T1_EXPORT permit 10
  match ip address prefix-list T1_PREFIXES

vrf context Tenant-1
  address-family ipv4 unicast
    import vrf advertise-vpn
    import vrf Tenant-3 map T1_IMPORT
    export vrf Tenant-3 map T1_EXPORT allow-vpn
```

Exact commands vary by NX-OS release. The policy intent matters more than the particular syntax:

- Mark or match exported routes deterministically.
- Leak the default route only in the intended direction.
- Leak internal prefixes back toward the services VRF for return traffic.
- Prevent the imported route from being recursively re-exported and forming a loop.
- Verify both the local VRF RIB and the EVPN Type 5 advertisements.

### 18.3 Common route-leaking failures

If the route appears in BGP EVPN but not in the tenant RIB, inspect the RT import policy, route-map, next-hop resolution, and route type. If forward traffic works but replies fail, the services VRF or firewall probably lacks a return route. If a default route appears on unintended tenants, the export match is too broad.

## 19. Practical design checklist

Before deploying VXLAN EVPN, document these decisions:

### Underlay

- Routing protocol and ASN/area design.
- VTEP loopback allocation and advertisement.
- ECMP and failure-detection behavior.
- End-to-end MTU.
- Multicast versus ingress replication.

### Overlay

- VLAN/bridge-domain to L2 VNI mapping.
- VRF to L3 VNI mapping.
- RD uniqueness and RT import/export policy.
- Route-reflector placement and redundancy.
- Type 2 versus Type 5 route requirements.
- ARP/ND suppression support.

### Multihoming

- vPC, ESI multihoming, or another explicitly supported attachment model.
- Unique and deterministic ESI assignment.
- All-active versus single-active redundancy.
- DF election behavior and service granularity.
- Type 1 aliasing and fast-withdrawal behavior.
- Split-horizon and duplicate-BUM validation.
- Orphan-port behavior and failure recovery.

### Gateways and services

- Anycast gateway IP and MAC consistency.
- Symmetric versus asymmetric IRB.
- Firewall or load-balancer insertion.
- Shared-services and route-leaking policy.
- North-south default-route origination and withdrawal.

### Multi-Site

- Site IDs, BGW PIPs, and site VIPs.
- DCI and fabric link tracking.
- Stretched L2/L3 VNI allow-list.
- BUM replication mode and DF behavior.
- Route re-origination and loop prevention.
- Failure-domain and maintenance procedures.

## 20. Security, policy, and operational hardening

VXLAN expands a Layer 2 service across an IP network, so the trust boundary must include the VTEPs and underlay. RFC 7348 explicitly notes that MAC-over-IP increases the attack surface: a rogue system capable of injecting acceptable VXLAN traffic could spoof endpoints, capture traffic, or cause denial of service. [RFC 7348, Section 7](https://www.rfc-editor.org/rfc/rfc7348.html#section-7)

### 20.1 Underlay and VTEP protection

- Permit UDP/4789 only between authorized VTEP addresses; do not expose the NVE transport to user-facing or untrusted networks.
- Filter spoofed infrastructure source addresses at fabric boundaries.
- Authenticate routing protocols where supported and protect BGP sessions with appropriate peer controls and control-plane policing.
- Keep tenant and management traffic out of the infrastructure routing table.
- Use explicit infrastructure ACLs for routing, BFD, PIM, NTP/PTP, telemetry, and management protocols.
- Rate-limit or police traffic that can create control-plane or endpoint-learning pressure.

IPsec can authenticate and encrypt VXLAN over an untrusted IP transport, but it introduces key management, MTU, performance, and operational considerations. Cisco Multi-Site may alternatively support CloudSec on specific hardware and releases; consult the feature matrix before assuming encryption is available.

### 20.2 Endpoint and tenant controls

- Treat an EVPN-learned MAC/IP binding as reachability information, not proof that the endpoint is trustworthy.
- Apply DHCP snooping, IP source guard, dynamic ARP inspection, RA guard, or equivalent first-hop security only where the platform supports the feature with VXLAN EVPN.
- Limit unknown-unicast flooding when the application permits it.
- Use storm control and endpoint-move/duplicate detection to contain loops and faulty hosts.
- Make inter-VRF route leaking deny-by-default and permit only documented prefixes and service paths.
- Keep firewall insertion symmetric so stateful flows use the same policy context in both directions.

### 20.3 Scale budgets

Capacity planning must cover more than the advertised “16 million VNIs.” Real limits include:

- Local and remote MAC entries.
- IPv4 ARP and IPv6 ND entries.
- Type 2 host routes and Type 5 prefixes.
- L2 and L3 VNIs, VLANs, SVIs, and VRFs.
- NVE peers and ingress-replication fan-out.
- Multicast groups and hardware replication lists.
- ECMP next hops, ESI/DF state, and Multi-Site re-originated routes.
- TCAM regions consumed by suppression, ACL, QoS, and first-hop-security features.

Record both platform maximums and the smaller validated design limits. Test convergence near the intended scale, because a configuration that fits in hardware may still miss convergence or control-plane objectives.

### 20.4 Observability baseline

Before production, capture a known-good baseline for underlay routes, EVPN neighbor state, route counts by type, NVE peers, VNI state, MAC/ARP/ND counts, replication lists, hardware utilization, and Multi-Site tracking. Alert on deviations rather than waiting for endpoint complaints. Correlating a Type 2 or Type 5 route with its RT, VNI, BGP next hop, tunnel peer, and installed hardware entry is the core troubleshooting skill for this architecture.

## 21. Final mental model

The architecture becomes easier to reason about when reduced to five mappings:

```text
Local access VLAN/bridge domain -> L2 VNI
Tenant routing table           -> VRF
Tenant VRF                     -> L3 VNI
Endpoint MAC/IP                -> advertising VTEP (EVPN Type 2)
Tenant IP prefix               -> routing next hop (EVPN Type 5)
```

The underlay only delivers packets between tunnel endpoints. EVPN tells the VTEPs where endpoints, prefixes, and replication members live. VXLAN carries the resulting bridged or routed payload. Distributed anycast gateways keep east-west routing local, while Multi-Site border gateways deliberately break a large network into independent failure and control-plane domains.

![Course summary topology: multi-pod, Multi-Site, multi-tenant, and shared services](/posts/vxlan-evpn-architecture/architecture-summary.svg)

That separation of responsibilities is the central idea of VXLAN EVPN architecture: a simple routed fabric underneath, policy-rich tenant overlays above it, and explicit control points wherever scale or failure isolation requires another boundary.
