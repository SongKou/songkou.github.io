+++
title = 'Arista_VXLAN_EVPN_Lab'
date = 2026-07-21T00:30:00+08:00
draft = false
categories = ['Network']
tags = ['VXLAN', 'EVPN', 'Arista', 'Multicast', 'MLAG', 'IGMP', 'Network']
+++

A VXLAN fabric has to answer one question before any host can talk: **how does BUM traffic (Broadcast, Unknown unicast, Multicast) reach every remote VTEP?** There are only two data-plane answers — head-end replication (HER, also called ingress replication) or an IP multicast underlay — and two control-plane ways to drive each of them (static configuration or BGP EVPN).

This lab builds the same Arista fabric four times, once per combination:

| Section | Flood mode | Control plane | Result in this lab |
|---|---|---|---|
| 2 | HER (static flood list) | Flood-and-learn | Works |
| 3 | Multicast underlay (PIM-SM) | Flood-and-learn | Config correct, data plane unsupported on vEOS-lab |
| 4 | HER (dynamic flood list) | BGP EVPN (Type-3 IMET) | Works |
| 4.1 | Multicast underlay | BGP EVPN (IMET + PMSI) | Config correct, same vEOS-lab limitation |

All four sections above only get traffic between hosts in the **same** VLAN. Section 5 covers inter-VLAN routing (IRB) — the asymmetric and symmetric models, the two prerequisites that break it silently, and the MLAG-specific configuration it needs. Section 6 closes with the theory the lab keeps bumping into: HER vs multicast trade-offs, IGMP snooping, the differences between IGMPv1/v2/v3, and when an IGMP querier is required.

## 1. Lab setup and introduction

The lab runs in EVE-NG. All six switches are **Arista vEOS-lab 4.33.1.1F** nodes; the hosts (VPC1/VPC2/VPC3) are EVE-NG VPCS nodes, `Switch` is any L2-capable node that can form an LACP bond (in the CLI captures below it appears under its node name `Linux2`, an IOS-based switch at 192.168.20.100), and `Test_Server` hangs off Border for out-of-fabric tests.

![EVPN/VXLAN lab physical topology and IP addressing: Border on top as RP, two spines, an MLAG leaf pair plus a standalone leaf, and four hosts](/posts/arista-vxlan-bum-her-vs-multicast/lab-topology.svg)

The same topology as text:

```text
                        +----------------------+  Et1
                        |  Border  (mcast RP)  |------ Test_Server
                        |  Lo0 10.255.255.1    |
                        +----------------------+
                     Et5 /                  \ Et4
           10.0.101.0/31/                    \10.0.102.0/31
                    Et5/                      \Et4
            +-----------+                  +-----------+
            |  Spine1   |                  |  Spine2   |
            | Lo0 .11   |                  | Lo0 .12   |
            +-----------+                  +-----------+
            Et1 | Et2 | Et4              Et2 | Et1 | Et3
                |     |    \             /      |     \
                |     |     +----------------------+   \
                |     +-----------+    /        |   \   \
                |            \     \  /         |    \   \
            Et1 | Et2          \    \/          | Et1  \   \
        +-----------+        +-----------+      +-----------+
        |   Leaf1   |  Et8   |   Leaf2   |      |   Leaf3   |
        | Lo0 .21   |========| Lo0 .22   |      | Lo0 .23   |
        +-----------+ Po100  +-----------+      +-----------+
        Et3 |   Et4 \         / Et4             Et1 |    | Et2
            |        \       /                      |    |
         VPC1         Switch (Po20, LACP)        VPC2    VPC3
        vlan10        vlan20                    vlan20   vlan30

  Leaf1 + Leaf2 = MLAG pair (domain ekou_test), one logical VTEP
```

**Roles:**

- **Border** — pure underlay router; becomes the PIM RP in the multicast sections.
- **Spine1/Spine2** — IP transit only; become BGP route reflectors in the EVPN sections. They never carry VXLAN state.
- **Leaf1/Leaf2** — MLAG pair acting as one logical VTEP; the `Switch` host is dual-homed to them over LACP.
- **Leaf3** — standalone VTEP with two single-homed hosts.

### 1.1 IP addressing scheme

Loopbacks (all /32, all in OSPF area 0):

| Device | Loopback0 (router-id, BGP peering) | Loopback1 (VTEP source) |
|---|---|---|
| Border | 10.255.255.1 | — |
| Spine1 | 10.255.255.11 | — |
| Spine2 | 10.255.255.12 | — |
| Leaf1 | 10.255.255.21 | **10.255.255.112 (shared with Leaf2)** |
| Leaf2 | 10.255.255.22 | **10.255.255.112 (shared with Leaf1)** |
| Leaf3 | 10.255.255.23 | 10.255.255.113 |

The MLAG pair shares one VTEP address on `Loopback1`. Arista requires MLAG peers to present a **single anycast VTEP IP** so that remote VTEPs see one logical endpoint and the spines load-balance to whichever peer is alive. `Loopback0` stays unique per device for router-ids and (later) BGP peering.

Point-to-point links (all /31; convention: **spine side = .0, leaf/border side = .1**):

| Link | Subnet | Side A | Side B |
|---|---|---|---|
| Border – Spine1 | 10.0.101.0/31 | Spine1 Et5 = .0 | Border Et5 = .1 |
| Border – Spine2 | 10.0.102.0/31 | Spine2 Et4 = .0 | Border Et4 = .1 |
| Spine1 – Leaf1 | 10.0.11.0/31 | Spine1 Et1 = .0 | Leaf1 Et1 = .1 |
| Spine1 – Leaf2 | 10.0.12.0/31 | Spine1 Et2 = .0 | Leaf2 Et2 = .1 |
| Spine1 – Leaf3 | 10.0.13.0/31 | Spine1 Et4 = .0 | Leaf3 Et4 = .1 |
| Spine2 – Leaf1 | 10.0.21.0/31 | Spine2 Et2 = .0 | Leaf1 Et2 = .1 |
| Spine2 – Leaf2 | 10.0.22.0/31 | Spine2 Et1 = .0 | Leaf2 Et1 = .1 |
| Spine2 – Leaf3 | 10.0.23.0/31 | Spine2 Et3 = .0 | Leaf3 Et3 = .1 |

Overlay plan (identical on every leaf):

| VLAN | VNI | Anycast SVI | Multicast group (sections 3 / 4.1) | Attached hosts |
|---|---|---|---|---|
| 10 | 1010 | 192.168.10.1/24 | 239.1.1.10 | VPC1 (Leaf1 Et3) |
| 20 | 1020 | 192.168.20.1/24 | 239.1.1.20 | Switch (MLAG Po20), VPC2 (Leaf3 Et1) |
| 30 | 1030 | 192.168.30.1/24 | 239.1.1.30 | VPC3 (Leaf3 Et2) |

Shared values:

| Item | Value |
|---|---|
| Anycast gateway MAC | 00:1c:73:00:00:01 |
| MLAG domain | ekou_test |
| MLAG peer VLAN / SVI | VLAN 4094, 169.254.1.1/30 (Leaf1) and 169.254.1.2/30 (Leaf2) |
| MLAG peer-link | Port-Channel100 (Et8 on both peers) |
| VXLAN UDP port | 4789 |
| BGP AS (sections 4 / 4.1) | 65000 (iBGP, spines as route reflectors) |
| Host IPs | VPC1 .10.100, Switch/Linux2 .20.100, VPC2 .20.200, VPC3 .30.103 |

VLAN 20 is the interesting one: it is stretched between the MLAG logical VTEP and Leaf3, so any BUM scheme has to carry its ARP/broadcast across the fabric before `Switch` and VPC2 can exchange a single ping.

## 2. Baseline: infrastructure + VXLAN with static HER

This section builds everything the four flood modes share — OSPF underlay, MLAG, anycast gateways — and then brings up VXLAN in the simplest possible way: static head-end replication, where each leaf is manually told the list of remote VTEPs to which it must unicast-replicate every BUM frame.

All switches already have `service routing protocols model multi-agent` and `ip routing` (vEOS-lab 4.33 defaults to the multi-agent model, which is also mandatory for EVPN later).

### 2.1 OSPF underlay

Every fabric link is a routed /31 with OSPF network type point-to-point (skips DR/BDR election and speeds up adjacency). All loopbacks are injected into area 0.

Border:

```text
interface Ethernet4
   no switchport
   ip address 10.0.102.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet5
   no switchport
   ip address 10.0.101.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.255.255.1/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.255.255.1
   max-lsa 12000
```

Spine1 (Spine2 is the same pattern with its own interfaces/addresses from the table in 1.1):

```text
interface Ethernet1
   no switchport
   ip address 10.0.11.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   no switchport
   ip address 10.0.12.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet4
   no switchport
   ip address 10.0.13.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet5
   no switchport
   ip address 10.0.101.0/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.255.255.11/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.255.255.11
   max-lsa 12000
```

Leaf1 (Leaf2 mirrors it; Leaf3 uses Et3/Et4 as uplinks):

```text
interface Ethernet1
   no switchport
   ip address 10.0.11.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Ethernet2
   no switchport
   ip address 10.0.21.1/31
   ip ospf network point-to-point
   ip ospf area 0.0.0.0
!
interface Loopback0
   ip address 10.255.255.21/32
   ip ospf area 0.0.0.0
!
interface Loopback1
   ip address 10.255.255.112/32
   ip ospf area 0.0.0.0
!
ip routing
!
router ospf 1
   router-id 10.255.255.21
   max-lsa 12000
```

`Loopback1` is **identical on Leaf1 and Leaf2** (10.255.255.112) and unique on Leaf3 (10.255.255.113). Both MLAG peers advertise the same /32 into OSPF, so the spines install it as a two-way ECMP route — this is exactly what makes the anycast VTEP work: encapsulated traffic to 10.255.255.112 hashes to either peer, and either peer can decapsulate it.

Verification:

```text
show ip ospf neighbor          ! every P2P link FULL
show ip route 10.255.255.112   ! on a spine: two ECMP next-hops (Leaf1 and Leaf2)
show ip route ospf             ! all loopbacks and /31s present everywhere
ping 10.255.255.113 source Loopback1   ! VTEP-to-VTEP reachability from Leaf1
```

Do not continue until every loopback is pingable from every leaf — every flood mode in this post rides on this underlay.

### 2.2 MLAG on Leaf1/Leaf2

The MLAG pair uses VLAN 4094 over Port-Channel100 (Et8) as peer link and keepalive path. Identical on both peers except the SVI address and `peer-address`:

Leaf1:

```text
no spanning-tree vlan-id 4094
!
vlan 4094
   name MLAG_PEER
!
interface Port-Channel100
   description MLAG_PEER_LINK
   switchport mode trunk
!
interface Ethernet8
   channel-group 100 mode active
!
interface Vlan4094
   ip address 169.254.1.1/30
!
mlag configuration
   domain-id ekou_test
   local-interface Vlan4094
   peer-address 169.254.1.2
   peer-link Port-Channel100
```

Leaf2 differs only in `ip address 169.254.1.2/30` and `peer-address 169.254.1.1`.

The dual-homed member port toward `Switch` (both peers, identical `mlag` ID):

```text
vlan 10,20,30
!
interface Port-Channel20
   description MLAG_TO_LINUX_VLAN20
   switchport access vlan 20
   mlag 20
!
interface Ethernet4
   channel-group 20 mode active
```

Access ports for the single-homed hosts:

```text
! Leaf1
interface Ethernet3
   switchport access vlan 10
! Leaf3
interface Ethernet1
   switchport access vlan 20
interface Ethernet2
   switchport access vlan 30
```

Verification:

```text
show mlag                      ! State: Active, Peer-link up, negotiation status Connected
show mlag interfaces detail    ! mlag 20 active-full on both peers
show port-channel 20 detailed   ! LACP bundled on both members

Leaf1(config-if-Et1-2)#show mlag  
MLAG Configuration:              
domain-id                          :           ekou_test
local-interface                    :            Vlan4094
peer-address                       :         169.254.1.2
peer-link                          :     Port-Channel100
hb-peer-address                    :             0.0.0.0
peer-config                        :          consistent
                                                       
MLAG Status:                     
state                              :              Active
negotiation status                 :           Connected
peer-link status                   :                  Up
local-int status                   :                  Up
system-id                          :   52:00:00:d5:5d:c0
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False
                                                       
MLAG Ports:                      
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   1

Leaf1(config-if-Et1-2)#show mlag interfaces detail 
                                        local/remote                           
 mlag         state   local   remote    oper    config    last change   changes
------ ------------- ------- -------- ------- --------- --------------- -------
   20   active-full    Po20     Po20   up/up   ena/ena   21:28:14 ago         8
Leaf1(config-if-Et1-2)#show port-channel 20 detailed 
Port Channel Port-Channel20 (Fallback State: Unconfigured):
Minimum links: unconfigured
Minimum speed: unconfigured
Current weight/Max weight: 1/16
  Active Ports:
     Port            Time Became Active   Protocol   Mode      Weight   State  
    --------------- -------------------- ---------- --------- --------- -------
     Ethernet4       Sun 20:33:05         LACP       Active      1      Rx,Tx  
     PeerEthernet4   Sun 20:33:04         LACP       Active      0      Unknown

```

Two MLAG+VXLAN rules worth repeating because they fail silently otherwise:

- Both peers must have the **same VTEP IP** (our shared Loopback1) and the **same VLAN-to-VNI map**.
- Keep the peer link as a trunk carrying all VXLAN VLANs — single-homed hosts and failure cases depend on it.

### 2.3 Distributed anycast gateway

Every leaf carries the same three SVIs with `ip address virtual` and the same shared virtual MAC. A host's default gateway is therefore always its local leaf, no matter where it attaches or migrates:

```text
ip virtual-router mac-address 00:1c:73:00:00:01
!
interface Vlan10
   ip address virtual 192.168.10.1/24
!
interface Vlan20
   ip address virtual 192.168.20.1/24
!
interface Vlan30
   ip address virtual 192.168.30.1/24
```

`ip address virtual` (Arista's VARP-style anycast for VXLAN) means all leaves own the same IP **and** the same MAC for each SVI. Nothing is negotiated between leaves — unlike VRRP there is no master election and no hello traffic; every leaf simply answers ARP for the gateway locally.

Verification:

```text
show ip virtual-router         ! virtual MAC and the three virtual IPs listed

Leaf2#show ip virtual-router 
IP virtual router is configured with MAC address: 001c.7300.0001
IP virtual router address subnet routes not enabled
MAC address advertisement interval: 30 seconds
No interface with virtual IP address
```

From VPC1: `ping 192.168.10.1` must succeed before touching VXLAN — this only exercises the local leaf.

### 2.4 VXLAN with a static HER flood list

Now the actual overlay. In static HER, each VTEP keeps a configured list of remote VTEP IPs; every BUM frame is replicated at ingress into one unicast VXLAN packet per list entry. There are exactly two logical VTEPs in this fabric (MLAG pair = 10.255.255.112, Leaf3 = 10.255.255.113), so each side floods to exactly one remote address.

Leaf1 and Leaf2 (identical):

```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan flood vtep 10.255.255.113
```

Leaf3:

```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan flood vtep 10.255.255.112
```

`vxlan flood vtep` sets a global flood list for all VNIs; a per-VLAN variant (`vxlan vlan 20 flood vtep ...`) exists if you want to scope flooding to only the VLANs a remote VTEP actually serves. Note the MLAG pair lists **10.255.255.113**, not each other — the peer link handles intra-pair delivery, and a VTEP must never list its own address.

Verification:

```text
show interfaces vxlan 1        ! source interface Loopback1, VLAN-VNI map, flood mode: headend replication
show vxlan flood vtep          ! the configured remote VTEP per VLAN
show vxlan vtep                ! remote VTEPs known to the data plane
show mac address-table vlan 20 ! after traffic: remote MACs learned against interface Vxlan1
show vxlan address-table       ! same view from the VXLAN side, MAC -> remote VTEP binding
show vxlan config-sanity detail ! VTEP-IP mismatch, MLAG inconsistency, VLAN-VNI disagreement, in one shot
```

Output:

```text
Leaf3(config-if-Vx1)#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.255.255.113
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: CLI
  Remote MAC learning via Datapath
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 1010]        [20, 1020]        [30, 1030]       
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.255.255.112 
    20 10.255.255.112 
    30 10.255.255.112 
  Shared Router MAC is 0000.0000.0000
Leaf3(config-if-Vx1)#show vxlan flood vtep  
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10,20,30 *                      10.255.255.112 
* All VLANs in the indicated VLAN range list are using the default VTEP flood list 
Leaf3(config-if-Vx1)#show vxlan vtep   
Remote VTEPS for Vxlan1:

VTEP                 Tunnel Type(s)
-------------------- --------------
10.255.255.112       unicast, flood

Total number of remote VTEPS:  1
Leaf3(config-if-Vx1)#show mac address-table vlan 20
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  20    001c.7300.0001    STATIC      Cpu
  20    0050.7966.6807    DYNAMIC     Et1        1       0:00:36 ago
  20    5000.0008.0000    DYNAMIC     Vx1        1       0:00:41 ago
  20    5000.00d7.ee0b    DYNAMIC     Vx1        1       0:00:36 ago
Total Mac Addresses for this criterion: 4

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf3(config-if-Vx1)#show vxlan address-table   
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  0050.7966.6802  DYNAMIC   Vx1  10.255.255.112   1       0:00:42 ago
  20  5000.0008.0000  DYNAMIC   Vx1  10.255.255.112   1       0:00:48 ago
  20  5000.0008.0001  DYNAMIC   Vx1  10.255.255.112   1       0:00:03 ago
  20  5000.00d7.ee0b  DYNAMIC   Vx1  10.255.255.112   1       0:00:43 ago
Total Remote Mac Addresses for this criterion: 4


Leaf1(config-if-Vx1)#show interfaces vxlan 1
Vxlan1 is up, line protocol is up (connected)
  Hardware is Vxlan
  Source interface is Loopback1 and is active with 10.255.255.112
  Listening on UDP port 4789
  Replication/Flood Mode is headend with Flood List Source: CLI
  Remote MAC learning via Datapath
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 1010]        [20, 1020]        [30, 1030]       
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  Headend replication flood vtep list is:
    10 10.255.255.113 
    20 10.255.255.113 
    30 10.255.255.113 
  MLAG Shared Router MAC is 0000.0000.0000
Leaf1(config-if-Vx1)#show vxlan flood vtep  
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10,20,30 *                      10.255.255.113 
* All VLANs in the indicated VLAN range list are using the default VTEP flood list 
Leaf1(config-if-Vx1)#show vxlan vtep     
Remote VTEPS for Vxlan1:

VTEP                 Tunnel Type(s)
-------------------- --------------
10.255.255.113       flood, unicast

Total number of remote VTEPS:  1
Leaf1(config-if-Vx1)#show mac address-table vlan 20 
          Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  20    001c.7300.0001    STATIC      Cpu
  20    0050.7966.6807    DYNAMIC     Vx1        1       0:03:47 ago
  20    5000.0008.0000    DYNAMIC     Po20       1       4:16:34 ago
  20    5000.0008.0001    DYNAMIC     Po20       3       4:16:42 ago
  20    5000.0008.8014    DYNAMIC     Po20       1       0:02:06 ago
  20    5000.00d5.5dc0    STATIC      Po100
Total Mac Addresses for this criterion: 6

          Multicast Mac Address Table
------------------------------------------------------------------

Vlan    Mac Address       Type        Ports
----    -----------       ----        -----
Total Mac Addresses for this criterion: 0
Leaf1(config-if-Vx1)#show vxlan address-table 
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  10  5000.0072.8b31  DYNAMIC   Vx1  10.255.255.113   1       0:03:52 ago
  20  0050.7966.6807  DYNAMIC   Vx1  10.255.255.113   1       0:03:52 ago
Total Remote Mac Addresses for this criterion: 2


vpc2> ping 192.168.20.1  

84 bytes from 192.168.20.1 icmp_seq=1 ttl=64 time=3.138 ms
84 bytes from 192.168.20.1 icmp_seq=2 ttl=64 time=1.587 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=64 time=1.423 ms
84 bytes from 192.168.20.1 icmp_seq=4 ttl=64 time=1.305 ms
84 bytes from 192.168.20.1 icmp_seq=5 ttl=64 time=1.577 ms

vpc2> ping 192.168.20.100

host (192.168.20.100) not reachable

vpc2> ping 192.168.20.1  

84 bytes from 192.168.20.1 icmp_seq=1 ttl=64 time=1.159 ms
84 bytes from 192.168.20.1 icmp_seq=2 ttl=64 time=1.593 ms
84 bytes from 192.168.20.1 icmp_seq=3 ttl=64 time=1.257 ms
^C
vpc2> ping 192.168.20.100

host (192.168.20.100) not reachable

vpc2> ping 192.168.20.100

84 bytes from 192.168.20.100 icmp_seq=1 ttl=255 time=10.671 ms
84 bytes from 192.168.20.100 icmp_seq=2 ttl=255 time=8.776 ms
84 bytes from 192.168.20.100 icmp_seq=3 ttl=255 time=11.893 ms
84 bytes from 192.168.20.100 icmp_seq=4 ttl=255 time=9.070 ms
84 bytes from 192.168.20.100 icmp_seq=5 ttl=255 time=9.017 ms


vpc2>  show ip

NAME        : vpc2[1]
IP/MASK     : 192.168.20.200/24
GATEWAY     : 192.168.20.1
DNS         : 
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500
```

One detail in that capture is worth keeping honest: the first two `ping 192.168.20.100` attempts failed. They were taken while `Vxlan1` still pointed at the wrong source interface (`Loopback0` instead of `Loopback1`), so Leaf3 was flooding to 10.255.255.112 while the MLAG pair only decapsulated traffic addressed to their Loopback0 IPs. A VTEP only decapsulates VXLAN destined to its **configured source IP** — the symptom is exactly this: local MACs learn fine, `show vxlan address-table` stays empty on both ends. `show vxlan config-sanity detail` catches the mismatch immediately; after fixing the source interface, the same ping succeeds.

End-to-end tests:

- **L2 stretch:** VPC2 (192.168.20.200, on Leaf3) pings `Switch`/Linux2 (192.168.20.100, on the MLAG pair) — the capture above. The first ARP broadcast is head-end replicated to 10.255.255.112, the MLAG pair floods it to VLAN 20, and both sides learn each other's MAC against `Vxlan1`.
- **MLAG failover:** shut Leaf1's uplinks or the whole node; `Switch` traffic keeps flowing through Leaf2 because the spines still have a path to 10.255.255.112.
- **Inter-VLAN (VPC1 → VPC3)?** Not yet — do not expect it to work at this stage. Routing between VNIs is a separate feature with its own prerequisites (including the virtual VTEP that `show vxlan config-sanity` is already warning about); section 5 builds it properly.

This all works on vEOS-lab. The obvious drawback: every time a VTEP is added, **every other VTEP's flood list must be edited by hand**. Two VTEPs make it trivial; two hundred make it an outage generator. That is the operational hole that either multicast (section 3) or EVPN (section 4) fills.

## 3. VXLAN with a multicast underlay (and why vEOS-lab can't do it)

The second data-plane option replaces the unicast flood list with underlay IP multicast: each VNI maps to a group, every VTEP joins the groups of its local VNIs, and a BUM frame is encapsulated **once** toward the group address. The underlay multicast tree — built by PIM-SM — does all replication.

### 3.1 Configuration

On every leaf, the `Vxlan1` flood list is replaced by group mappings (this is the configuration that was actually captured in the lab):

```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan vlan 10 multicast group 239.1.1.10
   vxlan vlan 20 multicast group 239.1.1.20
   vxlan vlan 30 multicast group 239.1.1.30
   no vxlan flood vtep 10.255.255.113    ! remove the static list from section 2.4
```

Multicast routing and PIM sparse mode on **every device** (leaves, spines, Border). vEOS-lab needs the explicit kernel software-forwarding knob:

```text
router multicast
   ipv4
      routing
      software-forwarding kernel
!
router pim sparse-mode
   ipv4
      rp address 10.255.255.1
```

PIM on every fabric interface (shown for Spine1; same two lines go on all /31 interfaces of all devices):

```text
interface Ethernet1
   pim ipv4 sparse-mode
interface Ethernet2
   pim ipv4 sparse-mode
interface Ethernet4
   pim ipv4 sparse-mode
interface Ethernet5
   pim ipv4 sparse-mode
```

Border is the rendezvous point simply because its Loopback0 (10.255.255.1) is what every router points at with `rp address`. A static RP is fine for a lab; production would use anycast RP on a pair of devices (typically the spines) with MSDP, or PIM BSR/Auto-RP for dynamic election.

### 3.2 What the control plane shows

PIM itself comes up perfectly. Real output from Spine1:

```text
SP1#show ip pim neighbor
PIM Neighbor Table for default VRF
Neighbor Address  Interface  Uptime    Expires   Mode    Transport
10.0.11.1         Ethernet1  00:16:28  00:01:19  sparse  datagram
10.0.12.1         Ethernet2  00:14:48  00:01:27  sparse  datagram
10.0.13.1         Ethernet4  00:15:36  00:01:40  sparse  datagram
10.0.101.1        Ethernet5  00:18:05  00:01:23  sparse  datagram

SP1#show ip pim rp
Group: 224.0.0.0/4
  RP: 10.255.255.1
    Uptime: 0:18:43, Expires: never, Priority: 0, Override: False
```

Every device sees all its PIM neighbors and agrees on the RP. On working hardware, the next step would be each VTEP joining its VNI groups, which shows up as `(*, 239.1.1.20)` entries rooted at the RP. Instead, on every single node:

```text
Leaf1#show ip mroute
PIM Sparse Mode Multicast Routing Table
Flags: E - Entry forwarding on the RPT, J - Joining to the SPT
    ...
(no entries)


Leaf1#show interfaces vxlan1
Vxlan1 is down, line protocol is down (notconnect)
  Hardware is Vxlan
  Source interface is Loopback0 and is active with 10.255.255.21
  Listening on UDP port 4789
  Replication/Flood Mode is not initialized yet
  Remote MAC learning via Datapath
  VNI mapping to VLANs
  Static VLAN to VNI mapping is 
    [10, 1010]        [20, 1020]        [30, 1030]       
  Note: All Dynamic VLANs used by VCS are internal VLANs.
        Use 'show vxlan vni' for details.
  Static VRF to VNI mapping is not configured
  MLAG Shared Router MAC is 0000.0000.0000
```

The multicast routing table stays **empty on all six devices**. The leaves never originate a join for 239.1.1.10/20/30, `Vxlan1` reports `Replication/Flood Mode is not initialized yet` and stays down, and any traffic that needs BUM — starting with the very first ARP between `Switch` and VPC2 — dies at the ingress VTEP. (This capture was taken before the lab moved the VXLAN source to `Loopback1`, hence `Loopback0` in the output; the flood-mode and mroute behavior is identical either way.)

### 3.3 Why this cannot proceed in EVE-NG

Nothing is wrong with the configuration — the problem is the platform:

- **vEOS-lab forwards in software.** It is a control-plane simulator: the EOS agents (OSPF, PIM, BGP, MLAG) are the real thing, but packet forwarding is done by the Linux kernel instead of a switching ASIC.
- **VXLAN multicast flood mode is a data-plane feature** that vEOS-lab's kernel path does not implement. The VTEP would need to (a) signal IGMP/PIM membership for each VNI group, (b) encapsulate BUM frames toward the group, and (c) decapsulate multicast-delivered VXLAN packets. None of that is wired up in the vEOS-lab forwarding path — which is exactly why the leaves never even send joins: the component that would drive them doesn't exist here.
- The `router multicast ... software-forwarding kernel` knob enables plain multicast **routing** in vEOS, but it does not extend to the VXLAN encap/decap replication path.

So the symptom pattern to recognize: **PIM neighbors up + RP known + mroute table permanently empty + VXLAN flood mode never initialized = platform limitation, not misconfiguration.** On hardware EOS (7050X/7060X/7280R and friends) the identical configuration builds `(*,G)` and `(S,G)` state and works. If you need to see VXLAN over multicast actually forward in a virtual lab, this is not achievable with vEOS-lab in EVE-NG; keep the configs as a hardware reference and use HER for anything you want to demonstrate live.

The lab therefore rolls back to a working state — and instead of returning to static flood lists, it moves to the modern answer: EVPN.

## 4. VXLAN EVPN with head-end replication

BGP EVPN replaces both weak spots of section 2: flood lists are **learned** instead of configured (EVPN Type-3, Inclusive Multicast Ethernet Tag), and MAC addresses are **advertised** instead of flood-learned (EVPN Type-2, MAC/IP). The data plane is still HER — ingress unicast replication — but the replication list now maintains itself. This is the mode the lab runs permanently, and everything in this section was verified working on vEOS-lab.

Design: iBGP AS 65000, spines as route reflectors, peering on Loopback0, EVPN address family only (`no bgp default ipv4-unicast` — the underlay stays OSPF's job). RD is unique per leaf (`<Loopback0>:<vlan>`), RT is fabric-wide per VNI (`1:<vlan>`).

### Step 1 — Clean up the multicast attempt

On all leaves:

```text
interface Vxlan1
   no vxlan vlan 10 multicast group 239.1.1.10
   no vxlan vlan 20 multicast group 239.1.1.20
   no vxlan vlan 30 multicast group 239.1.1.30
!
no router pim sparse-mode
no router multicast
```

(Remove `pim ipv4 sparse-mode` from the interfaces and the RP config on Border too, unless you're keeping them for section 4.1.)

### Step 2 — Spines as EVPN route reflectors

Spine1 (Spine2 identical except `router-id 10.255.255.12` and spine-peer 10.255.255.11):

```text
service routing protocols model multi-agent
!
router bgp 65000
   router-id 10.255.255.11
   no bgp default ipv4-unicast
   !
   neighbor EVPN-RRC peer group
   neighbor EVPN-RRC remote-as 65000
   neighbor EVPN-RRC update-source Loopback0
   neighbor EVPN-RRC send-community extended
   neighbor EVPN-RRC route-reflector-client
   !
   neighbor 10.255.255.21 peer group EVPN-RRC
   neighbor 10.255.255.22 peer group EVPN-RRC
   neighbor 10.255.255.23 peer group EVPN-RRC
   !
   neighbor 10.255.255.12 remote-as 65000
   neighbor 10.255.255.12 update-source Loopback0
   neighbor 10.255.255.12 send-community extended
   !
   address-family evpn
      neighbor EVPN-RRC activate
      neighbor 10.255.255.12 activate
```

Spine2:

```text
service routing protocols model multi-agent
!
router bgp 65000
   router-id 10.255.255.12
   no bgp default ipv4-unicast
   !
   neighbor EVPN-RRC peer group
   neighbor EVPN-RRC remote-as 65000
   neighbor EVPN-RRC update-source Loopback0
   neighbor EVPN-RRC send-community extended
   neighbor EVPN-RRC route-reflector-client
   !
   neighbor 10.255.255.21 peer group EVPN-RRC
   neighbor 10.255.255.22 peer group EVPN-RRC
   neighbor 10.255.255.23 peer group EVPN-RRC
   !
   neighbor 10.255.255.11 remote-as 65000
   neighbor 10.255.255.11 update-source Loopback0
   neighbor 10.255.255.11 send-community extended
   !
   address-family evpn
      neighbor EVPN-RRC activate
      neighbor 10.255.255.11 activate
```

`send-community extended` is not optional — route targets are extended communities, and without it the leaves receive routes they can never import. If `service routing protocols model multi-agent` was not already active, it requires a reload.

### Step 3 — MLAG leaves (Leaf1/Leaf2)

Everything VXLAN-facing is identical on both peers (shared Loopback1, same VLAN-VNI map, same RTs); only `router-id` and the RDs are per-leaf. Leaf1 shown:

Leaf1:
```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
!
router bgp 65000
   router-id 10.255.255.21
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   !
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN send-community extended
   neighbor 10.255.255.11 peer group EVPN
   neighbor 10.255.255.12 peer group EVPN
   !
   vlan 10
      rd 10.255.255.21:10
      route-target both 1:10
      redistribute learned
   vlan 20
      rd 10.255.255.21:20
      route-target both 1:20
      redistribute learned
   vlan 30
      rd 10.255.255.21:30
      route-target both 1:30
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
```

Leaf2:
```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
!
router bgp 65000
   router-id 10.255.255.22
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   !
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN send-community extended
   neighbor 10.255.255.11 peer group EVPN
   neighbor 10.255.255.12 peer group EVPN
   !
   vlan 10
      rd 10.255.255.22:10
      route-target both 1:10
      redistribute learned
   vlan 20
      rd 10.255.255.22:20
      route-target both 1:20
      redistribute learned
   vlan 30
      rd 10.255.255.22:30
      route-target both 1:30
      redistribute learned
   !
   address-family evpn
      neighbor EVPN activate
```


`redistribute learned` is what turns locally learned MACs (and ARP entries) into Type-2 advertisements. MLAG config, access ports, and anycast gateways from section 2 stay exactly as they were.

### Step 4 — Standalone leaf (Leaf3)

Same pattern with its own identifiers:

```text
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
!
router bgp 65000
   router-id 10.255.255.23
   no bgp default ipv4-unicast
   maximum-paths 4 ecmp 4
   neighbor EVPN peer group
   neighbor EVPN remote-as 65000
   neighbor EVPN update-source Loopback0
   neighbor EVPN send-community extended
   neighbor 10.255.255.11 peer group EVPN
   neighbor 10.255.255.12 peer group EVPN
   vlan 10
      rd 10.255.255.23:10
      route-target both 1:10
      redistribute learned
   vlan 20
      rd 10.255.255.23:20
      route-target both 1:20
      redistribute learned
   vlan 30
      rd 10.255.255.23:30
      route-target both 1:30
      redistribute learned
   address-family evpn
      neighbor EVPN activate
```

### Step 5 — Verification

```text
show bgp evpn summary            ! 2 sessions per leaf (both spines), Established, prefixes exchanged
show bgp evpn route-type imet    ! one Type-3 per (VTEP, VNI): 2 VTEPs x 3 VNIs = 6 IMET routes
show bgp evpn route-type mac-ip  ! Type-2 for every host MAC (and IP once ARP'd)
show vxlan vtep                  ! remote VTEP discovered via EVPN, not config
show vxlan flood vtep            ! HER flood list per VLAN, built from IMET routes
show vxlan address-table         ! remote MACs installed from Type-2, marked EVPN
show mlag                        ! still Active - EVPN and MLAG coexist
show vxlan config-sanity detail  ! VTEP-IP mismatch, MLAG inconsistency, VLAN-VNI disagreement, in one shot
```

`show vxlan config-sanity detail` is the single most useful command in this whole post. It compares the local VTEP config against the MLAG peer and against what the remote VTEPs are advertising, and it catches the two mistakes that cost the most time in this lab: a VXLAN source interface that doesn't match what remote VTEPs are addressing, and the missing virtual VTEP IP covered in section 5.

```text

Leaf1(config-router-bgp-af)#show bgp evpn summary  
BGP summary information for VRF default
Router identifier 10.255.255.21, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.255.255.11 4 65000             16        10    0    0 00:02:20 Estab   8      8
  10.255.255.12 4 65000             12        10    0    0 00:02:20 Estab   8      8
Leaf1(config-router-bgp-af)#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.255.255.21, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:10 imet 10.255.255.112
                                 -                     -       -       0       i
 * >      RD: 10.255.255.21:20 imet 10.255.255.112
                                 -                     -       -       0       i
 * >      RD: 10.255.255.21:30 imet 10.255.255.112
                                 -                     -       -       0       i
          RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
          RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
          RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 * >Ec    RD: 10.255.255.23:10 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.23:10 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.23:20 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.23:20 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.23:30 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.23:30 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.11 10.255.255.12 
Leaf1(config-router-bgp-af)# show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.255.21, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11


Leaf3(config-router-bgp-af)#show bgp evpn summary  
BGP summary information for VRF default
Router identifier 10.255.255.23, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.255.255.11 4 65000             15         7    0    0 00:00:25 Estab   10     10
  10.255.255.12 4 65000             12         7    0    0 00:00:25 Estab   10     10
Leaf3(config-router-bgp-af)#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.255.255.23, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.255.21:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.21:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.21:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 *  ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 *  ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >      RD: 10.255.255.23:10 imet 10.255.255.113
                                 -                     -       -       0       i
 * >      RD: 10.255.255.23:20 imet 10.255.255.113
                                 -                     -       -       0       i
 * >      RD: 10.255.255.23:30 imet 10.255.255.113
                                 -                     -       -       0       i
Leaf3(config-router-bgp-af)#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.255.23, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 10.255.255.12 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
Leaf3(config-router-bgp-af)#show vxlan vtep     
Remote VTEPS for Vxlan1:

VTEP                 Tunnel Type(s)
-------------------- --------------
10.255.255.112       unicast, flood

Total number of remote VTEPS:  1
Leaf3(config-router-bgp-af)#show vxlan flood vtep  
          VXLAN Flood VTEP Table
--------------------------------------------------------------------------------

VLANS                            Ip Address
-----------------------------   ------------------------------------------------
10,20,30                        10.255.255.112 
Leaf3(config-router-bgp-af)#show vxlan address-table 
          Vxlan Mac Address Table
----------------------------------------------------------------------

VLAN  Mac Address     Type      Prt  VTEP             Moves   Last Move
----  -----------     ----      ---  ----             -----   ---------
  20  5000.0008.0000  EVPN      Vx1  10.255.255.112   1       0:02:27 ago
  20  5000.0008.0001  EVPN      Vx1  10.255.255.112   1       0:02:27 ago
Total Remote Mac Addresses for this criterion: 2
Leaf3(config-router-bgp-af)#show mlag 
MLAG Configuration:              
domain-id                          :                   
local-interface                    :                   
peer-address                       :             0.0.0.0
peer-link                          :                   
hb-peer-address                    :             0.0.0.0
peer-config                        :                   
                                                       
MLAG Status:                     
state                              :            Disabled
negotiation status                 :                   
peer-link status                   :                   
local-int status                   :                   
system-id                          :   00:00:00:00:00:00
dual-primary detection             :            Disabled
dual-primary interface errdisabled :               False
                                                       
MLAG Ports:                      
Disabled                           :                   0
Configured                         :                   0
Inactive                           :                   0
Active-partial                     :                   0
Active-full                        :                   0


SP2(config-router-bgp)#show bgp evpn summary  
BGP summary information for VRF default
Router identifier 10.255.255.12, local AS number 65000
Neighbor Status Codes: m - Under maintenance
  Neighbor      V AS           MsgRcvd   MsgSent  InQ OutQ  Up/Down State   PfxRcd PfxAcc
  10.255.255.11 4 65000             19        27    0    0 00:05:08 Estab   5      5
  10.255.255.21 4 65000             10        12    0    0 00:02:11 Estab   5      5
  10.255.255.22 4 65000              9        15    0    0 00:01:03 Estab   5      5
  10.255.255.23 4 65000              7        12    0    0 00:00:30 Estab   3      3
SP2(config-router-bgp)#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.255.255.12, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.21:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.21:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.23:10 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
 * >      RD: 10.255.255.23:20 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
 * >      RD: 10.255.255.23:30 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
SP2(config-router-bgp)#show bgp evpn route-type imet
BGP routing table information for VRF default
Router identifier 10.255.255.12, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.21:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.21:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:10 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:30 imet 10.255.255.112
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.23:10 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
 * >      RD: 10.255.255.23:20 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
 * >      RD: 10.255.255.23:30 imet 10.255.255.113
                                 10.255.255.113        -       100     0       i
SP2(config-router-bgp)#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.255.12, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i
SP2(config-router-bgp)#show vxlan vtep     
SP2(config-router-bgp)#show bgp summary 
BGP summary information for VRF default
Router identifier 10.255.255.12, local AS number 65000
Neighbor               AS Session State AFI/SAFI                AFI/SAFI State   NLRI Rcd   NLRI Acc
------------- ----------- ------------- ----------------------- -------------- ---------- ----------
10.255.255.11       65000 Established   L2VPN EVPN              Negotiated             15         15
10.255.255.21       65000 Established   L2VPN EVPN              Negotiated              9          9
10.255.255.22       65000 Established   L2VPN EVPN              Negotiated              9          9
10.255.255.23       65000 Established   L2VPN EVPN              Negotiated              5          5
```

What to look for, and what the lab confirmed:

- `Vxlan1` comes up in **head-end replication flood mode with no `vxlan flood vtep` configured anywhere** — the flood lists in `show vxlan flood vtep` are exactly the remote VTEPs from the Type-3 routes. Add a fourth VTEP tomorrow and every flood list updates itself.
- The MLAG pair originates its Type-3/Type-2 routes with next-hop **10.255.255.112** from both peers; remote leaves see one logical VTEP with two BGP paths.
- Type-2 routes carrying MAC+IP enable **ARP suppression**: once VPC2's binding is known fabric-wide, a later ARP request for it is answered locally by the ingress leaf instead of being flooded at all — the first BUM-reduction win of EVPN before even touching multicast.
- Same-subnet (L2) tests pass on vEOS-lab: `Switch` ↔ VPC2 across the stretched VLAN 20, and MLAG failover.

Note what is **not** yet working at this point: hosts in **different** VLANs still cannot reach each other. Everything built so far is Layer 2 within a VNI plus a local default gateway. Bridging across the fabric and routing across the fabric are separate problems, and the second one needs its own configuration — that is section 5.

### 4.1 VXLAN EVPN with a multicast underlay

The fourth combination keeps the EVPN control plane and swaps the BUM data plane back to underlay multicast. EVPN still advertises Type-3 IMET routes, but they now carry a **PMSI Tunnel Attribute** announcing "reach my BUM via group 239.1.1.x" instead of "ingress-replicate to my VTEP IP". You get EVPN's MAC advertisement and ARP suppression **plus** single-copy BUM forwarding — the standard design for very large fabrics or multicast-heavy tenants.

Configuration on top of section 4: re-add the group mappings to `Vxlan1` on every leaf, and restore the PIM underlay from section 3 (RP on Border, `pim ipv4 sparse-mode` on every fabric link, `router multicast` everywhere):

```text
interface Vxlan1
   vxlan vlan 10 multicast group 239.1.1.10
   vxlan vlan 20 multicast group 239.1.1.20
   vxlan vlan 30 multicast group 239.1.1.30
!
router multicast
   ipv4
      routing
!
router pim sparse-mode
   ipv4
      rp address 10.255.255.1
```

Verification (in this order — control plane first, then data plane):

```text
show bgp evpn route-type imet detail   ! PMSI tunnel attribute now shows the multicast group per VNI
show ip pim neighbor                   ! unchanged from section 3 - all neighbors up
show ip mroute                         ! hardware: (*,G)/(S,G) per VNI group with Vxlan1 in the OIL
show vxlan flood vtep                  ! flood mode multicast; no unicast HER list needed
show interfaces vxlan 1                ! flood mode: multicast group
```

On vEOS-lab the result splits exactly along the control/data boundary: the IMET routes with their PMSI attributes are advertised and received correctly, PIM neighbors are all up — and the mroute table stays empty for the same reason as section 3. The data-plane half of this mode is the same unsupported VXLAN multicast replication path, so BUM forwarding never starts. **Type-2-driven known-unicast still works**, which produces a deceptive half-alive fabric: hosts whose MACs are already in EVPN can ping, but any flow that needs an initial broadcast fails. Verify this mode on hardware; check the exact EOS release notes for EVPN-multicast-underlay support on your platform before designing around it.

## 5. Inter-VLAN routing (IRB)

Up to this point the fabric only bridges. VPC1 (VLAN 10) can reach anything else in VLAN 10, `Switch` and VPC2 share VLAN 20 across two VTEPs, and every host can ping its own default gateway because the anycast SVI is local. But VPC1 still cannot reach VPC3 in VLAN 30, and that is not a bug in the previous sections — routing between VNIs is a separate feature with its own requirements.

The anycast gateway from section 2.3 solves only the *first hop*. It gives every host a gateway on its own leaf so traffic never trombones to a central router. What it does not do by itself is tell the fabric how a packet gets from VNI 1010 to VNI 1030 on a **different** VTEP.

### 5.1 The two IRB models

Arista implements both models from the IETF EVPN IRB draft, and the [EOS Integrated Routing and Bridging guide](https://www.arista.com/en/um-eos/eos-integrated-routing-and-bridging) defines them precisely.

**Asymmetric IRB** — "the inter-subnet routing functionality is performed by the ingress VTEP, with the packet after the routing action being VXLAN bridged to the destination VTEP." The egress VTEP "only then needs to remove the VXLAN header and forward the packet onto the local Layer 2 domain based on the VNI to VLAN mapping."

**Symmetric IRB** — "the ingress VTEP routes the traffic between the local subnet and the IP-VRF, which both VTEPs are a member of; the egress VTEP then routes the frame from the IP-VRF to the destination subnet." That IP-VRF has its own transit **L3 VNI**.

| | Asymmetric IRB | Symmetric IRB |
|---|---|---|
| Routing happens | ingress VTEP only | both ingress and egress VTEP |
| VNI used in flight | the **destination** VLAN's L2 VNI | a dedicated **L3 VNI** (IP-VRF transit) |
| Every leaf must host | **all** tenant VLANs, VNIs and anycast SVIs | only its **locally attached** subnets |
| Every leaf must learn | all tenant MACs **and** ARP bindings, everywhere | local hosts + remote prefixes |
| Tenant VRF required | no — works in the default VRF | yes |
| Control plane | works with flood-and-learn **or** EVPN | **EVPN only** |
| Scales | poorly — state grows with the whole tenant | well — state grows with what you actually host |
| Good for | small fabrics, labs, "every VLAN is everywhere" | production multi-tenant fabrics |

Arista's wording on the asymmetric requirement is worth quoting because it is the constraint people trip over: "the VTEP needs to be member of all the tenant's subnets/VNI and have an associated SVI with anycast IP for all the subnets, and this will be required on all VTEPs participating in the routing functionality for the tenant."

This lab already satisfies that — every leaf carries VLANs 10, 20 and 30 with identical anycast SVIs — so asymmetric IRB fits it naturally; sections 5.2 to 5.5 explain its mechanics and prerequisites. Section 5.6 then builds symmetric IRB, which — as 5.8 argues — is the one actually worth configuring, in the lab and everywhere else.

### 5.2 Two prerequisites that silently break asymmetric IRB

Before adding anything, two conditions have to be true on **every** leaf. Both fail by default in this topology.

**Prerequisite 1: the SVI must actually be operationally up.**

This is the one that catches most people in a lab. EOS applies **autostate** to SVIs by default, and the [EOS VLANs guide](https://www.arista.com/en/um-eos/eos-virtual-lans-vlans) states an SVI becomes active only when "the corresponding VLAN exists and is in the active state" **and** "one or more Layer 2 ports in the VLAN are up and in spanning-tree forwarding state."

Now look at the topology, and note that the MLAG peer-link matters here. Leaf1 and Leaf2 have access ports in VLANs 10 and 20 only, but `Port-Channel100` is a trunk carrying **all** VLANs including 30 — and a forwarding trunk port satisfies autostate just as well as an access port. So on the MLAG pair all three SVIs come up on their own.

**Leaf3 is the exposed one.** It is standalone, with no peer-link, and its only access ports are VLAN 20 (Et1) and VLAN 30 (Et2). Nothing on Leaf3 is in VLAN 10, so `Vlan10` has no member port and its SVI stays down under default autostate. Asymmetric IRB needs every SVI on every leaf, so that one down SVI breaks routing into VLAN 10 from Leaf3's side.

Arista's docs do not state whether a `vxlan vlan X vni Y` mapping counts as a member port for autostate, so do not rely on the VNI mapping to hold an SVI up. Check first, then fix what's actually down:

```text
show ip interface brief | include Vlan
```

The documented, guaranteed fix is explicit — and it is cheap to apply to every stretched SVI on every leaf rather than reasoning about which ones need it:

```text
interface Vlan10
   no autostate
interface Vlan20
   no autostate
interface Vlan30
   no autostate
```

Check it before anything else:

```text
show interfaces Vlan10 | include line protocol
show interfaces Vlan30 | include line protocol
show ip interface brief | include Vlan
```

Every SVI on every leaf must read `up/up`. An SVI that is down silently blackholes all routed traffic for that subnet.

**Prerequisite 2: routing must be enabled.** `ip routing` is off by default on EOS. It is already in the section 2 configs, but confirm with `show ip route` — if the connected `192.168.X.0/24` routes are missing, this is why.

### 5.3 Flood-and-learn fabric: Direct VXLAN Routing and the virtual VTEP

For the sections 2 and 3 fabric (static flood lists, no EVPN), Arista calls this model **Direct VXLAN Routing**. From the [EOS VXLAN Configuration guide](https://www.arista.com/en/um-eos/eos-vxlan-configuration): "Direct VXLAN routing with VXLAN enabled … configuring each VTEP with all VLANs. This allows packets to be VXLAN-bridged to a local VTEP and routed to remote VTEPs." Head-end replication is a stated prerequisite: "Head-end replication is required for VXLAN routing and to support VXLANs over MLAG." So the section 2 flood lists stay.

The piece that is missing is the **virtual VTEP (vVTEP)**, and `show vxlan config-sanity detail` says so directly:

```text
Category                            Result
---------------------------------- --------
Local VTEP Configuration Check       FAIL
  Loopback IP Address                 OK
  VLAN-VNI Map                        OK
  Flood List                          OK
  Routing                            FAIL   <-- Virtual VTEP IP is not configured
```

**Why it is needed.** Every leaf owns the same anycast gateway MAC `00:1c:73:00:00:01`. In a flood-and-learn fabric, MACs are learned from the data plane, so a remote VTEP that sees that MAC arriving from Leaf1's VTEP IP, then from Leaf3's VTEP IP, would keep re-binding the same MAC to a different VTEP — the gateway MAC flaps around the fabric and routed traffic follows it to the wrong place. The virtual VTEP fixes this by anchoring the shared virtual MAC behind one shared VTEP address that every routing VTEP presents.

**How it is configured.** There is no `vxlan virtual-vtep` command — the vVTEP is simply a **secondary address on the VXLAN source loopback**. The EOS VXLAN Configuration guide: "A virtual VTEP address is specified by configuring a secondary address on the loopback interface designated as the VXLAN's source interface," and "All VTEPs in the direct routing topology share the same virtual VTEP address."

So pick one new address — `10.255.255.110/32` — and put it on **all three leaves**, alongside each leaf's existing primary VTEP IP:

```text
! Leaf1 and Leaf2 (identical - shared MLAG VTEP plus shared vVTEP)
interface Loopback1
   ip address 10.255.255.112/32
   ip address 10.255.255.110/32 secondary
   ip ospf area 0.0.0.0
```

```text
! Leaf3
interface Loopback1
   ip address 10.255.255.113/32
   ip address 10.255.255.110/32 secondary
   ip ospf area 0.0.0.0
```

Unlike the MLAG VTEP IP — which is shared only by Leaf1 and Leaf2 — the vVTEP is shared by **every** leaf that routes. It is the one address in the fabric that is deliberately identical everywhere.

Then re-run the check:

```text
show vxlan config-sanity detail    ! Local VTEP Configuration Check -> Routing should now be OK
show ip route 10.255.255.110       ! on a spine: ECMP to all three leaves
```

Updated addressing table for this section:

| Device | Loopback0 (router-id) | Loopback1 primary (VTEP) | Loopback1 secondary (vVTEP) |
|---|---|---|---|
| Leaf1 | 10.255.255.21 | 10.255.255.112 (shared) | 10.255.255.110 |
| Leaf2 | 10.255.255.22 | 10.255.255.112 (shared) | 10.255.255.110 |
| Leaf3 | 10.255.255.23 | 10.255.255.113 | 10.255.255.110 |

### 5.4 MLAG: the shared router MAC

The `show interfaces vxlan 1` output in section 2 contained a line that was easy to skip past:

```text
MLAG Shared Router MAC is 0000.0000.0000
```

All-zeros means the feature is not enabled. On an MLAG pair doing VXLAN routing, each peer would otherwise use its own system MAC as the router MAC. Routed VXLAN traffic returning to the pair is ECMP-hashed to *either* peer, so a packet whose inner destination MAC belongs to Leaf1 can easily land on Leaf2, which then has to shunt it across the peer-link.

One command on `interface Vxlan1`, identical on **both** MLAG peers, fixes it:

```text
interface Vxlan1
   vxlan virtual-router encapsulation mac-address mlag-system-id
```

Per Arista's [Symmetric IRB with MLAG lab guide](https://labguides.testdrive.arista.com/2025.1/data_center/l2_l3_evpn_symm_mlag/), this "instructs the device to use the shared MLAG System ID as the router MAC when performing VXLAN routing operations and ensures that whichever switch in the MLAG receives the VXLAN Routed packet can provide forwarding of that traffic without shunting it over the MLAG peer-link." Arista documents it as required for symmetric IRB with MLAG.

Leaf3 is standalone and does not take this command. Verify:

```text
show interfaces vxlan 1 | include Shared Router MAC   ! now a real MAC, e.g. 021c.73xx.xxxx
```

Both peers must show the **same** value, and — importantly — that value must be **unique per MLAG pair** in the fabric. Arista's KB on common EVPN VXLAN misconfigurations warns that "if 2 or more MLAG VTEP pairs are configured with the same shared router MAC then EVPN Type-5 or Type-2 updates might get dropped and routes may also point to incorrect VTEP." With `mlag-system-id` that uniqueness is automatic, which is why it is preferable to hardcoding a MAC.

### 5.5 Asymmetric IRB on the EVPN fabric

On the section 4 EVPN fabric, asymmetric IRB needs **less** configuration than the flood-and-learn version, because EVPN removes the reason the vVTEP existed. Type-2 MAC/IP routes advertise host reachability explicitly, so the anycast gateway MAC is no longer learned from the data plane and cannot flap between VTEPs. Everything needed is already in place from sections 2 to 4, plus:

- `no autostate` on every stretched SVI (section 5.2) — still required.
- `vxlan virtual-router encapsulation mac-address mlag-system-id` on Leaf1 and Leaf2 (section 5.4) — still required.

That is it. No VRF, no L3 VNI, no additional BGP configuration. Because every leaf hosts every VLAN with an identical anycast SVI, each leaf can route locally into the destination VLAN and then bridge over that VLAN's existing L2 VNI.

Packet walk for VPC1 (192.168.10.100, VLAN 10, Leaf1) reaching VPC3 (192.168.30.103, VLAN 30, Leaf3):

1. VPC1 has no route to 192.168.30.0/24, so it sends the frame to its default gateway 192.168.10.1, destination MAC `00:1c:73:00:00:01`.
2. Leaf1 owns that MAC locally and routes. 192.168.30.0/24 is a connected subnet via its own `Vlan30` SVI — which is why that SVI must be up.
3. Leaf1 needs VPC3's MAC. With EVPN it already has it from a Type-2 route; in flood-and-learn it ARPs into VNI 1030, which is head-end replicated to Leaf3.
4. Leaf1 rewrites the frame into VLAN 30 and **bridges** it over VNI 1030 to 10.255.255.113.
5. Leaf3 decapsulates and bridges to Et2. It performs no routing at all.

The return path is the mirror image, and it crosses the fabric on **VNI 1010** instead — different VNI in each direction. That asymmetry is what names the model.

Verification:

```text
show ip route 192.168.30.0/24          ! connected via Vlan30 on every leaf
show ip arp                            ! remote host bindings present
show bgp evpn route-type mac-ip        ! EVPN only: Type-2 with both MAC and IP
show vxlan address-table               ! destination MAC bound to the remote VTEP
```

Then from VPC1: `ping 192.168.30.103`. If it fails, work the list in order — SVI up, `ip routing`, ARP resolved, MAC in the VXLAN table.

### 5.6 Symmetric IRB with an L3 VNI (EVPN only)

Asymmetric IRB works here because the lab is tiny and every leaf genuinely hosts every VLAN. That assumption collapses at scale: with 200 leaves and 500 VLANs, every leaf would have to hold every tenant SVI plus every tenant MAC and ARP entry. Symmetric IRB fixes that by routing into a tenant **IP-VRF** carried over a dedicated **L3 VNI**, so a leaf only needs the subnets it actually hosts.

This requires EVPN. The L3 VNI is advertised through EVPN Type-5 (and Type-2 with the L3 VNI and Router MAC extended communities); there is no flood-and-learn mechanism that carries IP-VRF prefixes between VTEPs, so this cannot be bolted onto the section 2 fabric.

**If you go straight to symmetric IRB, skip section 5.3 — you do not need the virtual VTEP.** The vVTEP exists only to anchor the shared anycast gateway MAC in a fabric that learns MACs from the data plane. EVPN advertises that MAC explicitly in Type-2 routes, so the flapping problem never arises. Arista's own Symmetric IRB with MLAG lab guide configures no secondary loopback address and still reports `Virtual VTEP IP: OK` in `show vxlan config-sanity`, which also suggests those `Routing` and `Virtual VTEP IP` rows key off the legacy direct-routing feature rather than EVPN symmetric IRB. Practical consequence: once you are on EVPN with an L3 VNI, do not chase a `Routing` FAIL in config-sanity — judge the fabric by `show ip route vrf TENANT_A` and an actual ping.

**The Arista-specific detail worth knowing up front:** unlike Cisco NX-OS, EOS does **not** require you to create a dedicated L3 VNI VLAN with an SVI. The `vxlan vrf <NAME> vni <VNI>` command binds the IP-VRF to the transit VNI directly and EOS allocates an internal VLAN for it automatically. You will see it in `show interfaces Vxlan1` as, for example, `Dynamic VLAN to VNI mapping for 'evpn' is [4092, 5001]`. If you come from NX-OS and go looking for where to put the L3VNI SVI, there isn't one.

The blocks below are complete for this step and given per device, so each one pastes straight into the right switch with no edits. They fold in the `no autostate` from 5.2 and the MLAG shared router MAC from 5.4, so you do not have to assemble them from three sections. They sit on top of the section 4 EVPN fabric (spines as route reflectors, per-leaf BGP with the L2 MAC-VRFs), which is unchanged. Only the L3 VNI in `interface Vxlan1` and the `vrf` block in `router bgp` are genuinely new. The spines need nothing — they never carry VXLAN or tenant state.

**Leaf1** — MLAG peer:

```text
! --- global (unchanged from section 2.3, shown for completeness) ---
ip virtual-router mac-address 00:1c:73:00:00:01
!
! --- tenant VRF ---
vrf instance TENANT_A
!
ip routing vrf TENANT_A
!
! --- tenant SVIs: vrf FIRST, then the address ---
interface Vlan10
   vrf TENANT_A
   ip address virtual 192.168.10.1/24
   no autostate
!
interface Vlan20
   vrf TENANT_A
   ip address virtual 192.168.20.1/24
   no autostate
!
interface Vlan30
   vrf TENANT_A
   ip address virtual 192.168.30.1/24
   no autostate
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan vrf TENANT_A vni 50000                              ! the L3 VNI
   vxlan virtual-router encapsulation mac-address mlag-system-id   ! MLAG pair only
!
router bgp 65000
   vrf TENANT_A
      rd 10.255.255.21:50000
      route-target import evpn 1:50000
      route-target export evpn 1:50000
      redistribute connected
```

**Leaf2** — MLAG peer. Identical to Leaf1 apart from the RD in the `router bgp` vrf block (`10.255.255.22:50000`):

```text
! --- global (unchanged from section 2.3, shown for completeness) ---
ip virtual-router mac-address 00:1c:73:00:00:01
!
! --- tenant VRF ---
vrf instance TENANT_A
!
ip routing vrf TENANT_A
!
! --- tenant SVIs: vrf FIRST, then the address ---
interface Vlan10
   vrf TENANT_A
   ip address virtual 192.168.10.1/24
   no autostate
!
interface Vlan20
   vrf TENANT_A
   ip address virtual 192.168.20.1/24
   no autostate
!
interface Vlan30
   vrf TENANT_A
   ip address virtual 192.168.30.1/24
   no autostate
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan vrf TENANT_A vni 50000                              ! the L3 VNI
   vxlan virtual-router encapsulation mac-address mlag-system-id   ! MLAG pair only
!
router bgp 65000
   vrf TENANT_A
      rd 10.255.255.22:50000
      route-target import evpn 1:50000
      route-target export evpn 1:50000
      redistribute connected
```

Everything except the RD **must** match between the two peers — the VLAN-to-VNI map, the L3 VNI, the anycast addresses and the shared router MAC command. A mismatch here is what `show vxlan config-sanity detail` and `show mlag config-sanity` exist to catch.

**Leaf3 — standalone.** Same shape, with two differences: no `vxlan virtual-router encapsulation` line (it has no MLAG peer), and `Vlan10` is optional:

```text
ip virtual-router mac-address 00:1c:73:00:00:01
!
vrf instance TENANT_A
!
ip routing vrf TENANT_A
!
interface Vlan20
   vrf TENANT_A
   ip address virtual 192.168.20.1/24
   no autostate
!
interface Vlan30
   vrf TENANT_A
   ip address virtual 192.168.30.1/24
   no autostate
!
! Vlan10 is optional here - Leaf3 hosts no VLAN 10 device, and symmetric IRB
! reaches VLAN 10 through the IP-VRF. Keeping it is harmless; omitting it is
! the point of the model. If you keep it, keep 'no autostate' with it.
!
interface Vxlan1
   vxlan source-interface Loopback1
   vxlan udp-port 4789
   vxlan vlan 10 vni 1010
   vxlan vlan 20 vni 1020
   vxlan vlan 30 vni 1030
   vxlan vrf TENANT_A vni 50000
!
router bgp 65000
   vrf TENANT_A
      rd 10.255.255.23:50000
      route-target import evpn 1:50000
      route-target export evpn 1:50000
      redistribute connected
```

Four things that bite:

- **`ip routing vrf TENANT_A` is mandatory.** A new VRF is created with inter-subnet routing disabled. Without this line the VRF exists and routes nothing.
- **Order matters inside the SVI.** Assigning `vrf TENANT_A` to an interface that already has an address clears that address. Set `vrf` first, then re-apply `ip address virtual` — which is why the blocks above are written in that order.
- **`no autostate` is belt-and-braces here, and worth keeping.** In the symmetric model a leaf only needs its local subnets, so most of these SVIs would stay up on their own. The case it protects against is a VLAN whose only member is a single host port: power off that host and the SVI drops, and the leaf silently stops routing for that subnet.
- **Do this on all leaves at once.** Moving the SVIs out of the default VRF on some leaves and not others splits the tenant in half. The underlay SVIs, loopbacks and MLAG peer VLAN 4094 stay in the default VRF — only the tenant SVIs move.

The L3 VNI (50000) uses its own route target, distinct from the per-VLAN L2 route targets configured in section 4. The L2 RTs move MAC reachability; the L3 RT moves prefix reachability. Both sets of route targets stay configured — symmetric IRB adds L3 reachability, it does not replace the L2 fabric.

Verification:

```text
show vrf                                  ! TENANT_A present, routing enabled
show ip route vrf TENANT_A                ! local connected subnets + remote subnets via VXLAN
show bgp evpn route-type ip-prefix ipv4   ! Type-5 prefix routes carrying the L3 VNI
show interfaces Vxlan1                     ! "Dynamic VLAN to VNI mapping" shows the auto-created L3VNI VLAN
show vxlan vni                             ! L2 VNIs plus the VRF-mapped L3 VNI
show bgp evpn route-type mac-ip           ! Type-2 mac-ip routing
```
Output:

```text


vpc1> ping 192.168.20.200

84 bytes from 192.168.20.200 icmp_seq=1 ttl=62 time=5.121 ms
84 bytes from 192.168.20.200 icmp_seq=2 ttl=62 time=6.291 ms
84 bytes from 192.168.20.200 icmp_seq=3 ttl=62 time=15.382 ms
84 bytes from 192.168.20.200 icmp_seq=4 ttl=62 time=5.087 ms
^C
vpc1> ping 192.168.20.100

84 bytes from 192.168.20.100 icmp_seq=1 ttl=254 time=3.370 ms
84 bytes from 192.168.20.100 icmp_seq=2 ttl=254 time=3.263 ms
84 bytes from 192.168.20.100 icmp_seq=3 ttl=254 time=2.616 ms
192.168.20.100 icmp_seq=4 timeout
84 bytes from 192.168.20.100 icmp_seq=5 ttl=254 time=6.732 ms

vpc1> show ip

NAME        : vpc1[1]
IP/MASK     : 192.168.10.100/24
GATEWAY     : 192.168.10.1
DNS         : 
MAC         : 00:50:79:66:68:02
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

vpc1> 


Linux2#ping 192.168.10.1
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.1, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 2/3/5 ms
Linux2#ping 192.168.10.100
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.10.100, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 3/6/12 ms
Linux2#ping 192.168.20.200
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.20.200, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 7/14/25 ms
Linux2#


vpc2>  show ip

NAME        : vpc2[1]
IP/MASK     : 192.168.20.200/24
GATEWAY     : 192.168.20.1
DNS         : 
MAC         : 00:50:79:66:68:07
LPORT       : 20000
RHOST:PORT  : 127.0.0.1:30000
MTU         : 1500

vpc2> ping 192.168.20.100

84 bytes from 192.168.20.100 icmp_seq=1 ttl=255 time=16.116 ms
84 bytes from 192.168.20.100 icmp_seq=2 ttl=255 time=6.766 ms
84 bytes from 192.168.20.100 icmp_seq=3 ttl=255 time=7.727 ms
84 bytes from 192.168.20.100 icmp_seq=4 ttl=255 time=10.210 ms
84 bytes from 192.168.20.100 icmp_seq=5 ttl=255 time=8.469 ms

vpc2> ping 192.168.10.1  

84 bytes from 192.168.10.1 icmp_seq=1 ttl=64 time=124.150 ms
84 bytes from 192.168.10.1 icmp_seq=2 ttl=64 time=8.681 ms
^C
vpc2> ping 192.168.10.100

84 bytes from 192.168.10.100 icmp_seq=1 ttl=62 time=11.000 ms
84 bytes from 192.168.10.100 icmp_seq=2 ttl=62 time=6.215 ms
84 bytes from 192.168.10.100 icmp_seq=3 ttl=62 time=5.763 ms
84 bytes from 192.168.10.100 icmp_seq=4 ttl=62 time=5.758 ms
84 bytes from 192.168.10.100 icmp_seq=5 ttl=62 time=4.766 ms

Leaf1(config)#  show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.255.21, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:10 mac-ip 0050.7966.6802
                                 -                     -       -       0       i
          RD: 10.255.255.22:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:10 mac-ip 0050.7966.6802 192.168.10.100
                                 -                     -       -       0       i
          RD: 10.255.255.22:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.8014
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:20 mac-ip 5000.0008.8014 192.168.20.100
                                 -                     -       -       0       i
          RD: 10.255.255.22:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
Leaf1(config)# show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.255.255.21, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >      RD: 10.255.255.21:50000 ip-prefix 192.168.10.0/24
                                 -                     -       -       0       i
          RD: 10.255.255.22:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >      RD: 10.255.255.21:50000 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
          RD: 10.255.255.22:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >Ec    RD: 10.255.255.23:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.23:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.11 
 * >      RD: 10.255.255.21:50000 ip-prefix 192.168.30.0/24
                                 -                     -       -       0       i
          RD: 10.255.255.22:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
          RD: 10.255.255.22:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >Ec    RD: 10.255.255.23:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.23:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.113        -       100     0       i Or-ID: 10.255.255.23 C-LST: 10.255.255.11 

Leaf3#show bgp evpn route-type mac-ip
BGP routing table information for VRF default
Router identifier 10.255.255.23, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.255.21:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.22:10 mac-ip 0050.7966.6802
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.21:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.22:10 mac-ip 0050.7966.6802 192.168.10.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0000
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.0001
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.8014
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 * >Ec    RD: 10.255.255.21:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 *  ec    RD: 10.255.255.22:20 mac-ip 5000.0008.8014 192.168.20.100
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
Leaf3# show bgp evpn route-type ip-prefix ipv4
BGP routing table information for VRF default
Router identifier 10.255.255.23, local AS number 65000
Route status codes: * - valid, > - active, S - Stale, E - ECMP head, e - ECMP
                    c - Contributing to ECMP, % - Pending best path selection
Origin codes: i - IGP, e - EGP, ? - incomplete
AS Path Attributes: Or-ID - Originator ID, C-LST - Cluster List, LL Nexthop - Link Local Nexthop

          Network                Next Hop              Metric  LocPref Weight  Path
 * >Ec    RD: 10.255.255.21:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.22:50000 ip-prefix 192.168.10.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.21:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.22:50000 ip-prefix 192.168.20.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >      RD: 10.255.255.23:50000 ip-prefix 192.168.20.0/24
                                 -                     -       -       0       i
 * >Ec    RD: 10.255.255.21:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.21:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.21 C-LST: 10.255.255.11 
 * >Ec    RD: 10.255.255.22:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.12 
 *  ec    RD: 10.255.255.22:50000 ip-prefix 192.168.30.0/24
                                 10.255.255.112        -       100     0       i Or-ID: 10.255.255.22 C-LST: 10.255.255.11 
 * >      RD: 10.255.255.23:50000 ip-prefix 192.168.30.0/24
                                 -                     -       -       0       i
```

In the routed path now, the packet leaves Leaf1 encapsulated in **VNI 50000** rather than VNI 1030, and Leaf3 routes it out of the IP-VRF into VLAN 30. Both directions use the same L3 VNI — hence "symmetric".

### 5.7 Verifying which model you are actually running

Both models produce working pings between the same pair of hosts, so "it works" does not tell you which one you built. Five checks do, in rough order of how fast they are.

**1. Count the TTL decrements.** The fastest test, and it needs no switch access at all — just ping between two hosts in different VLANs and read the TTL:

```text
vpc1> ping 192.168.20.200
84 bytes from 192.168.20.200 icmp_seq=1 ttl=62
```

VPCS sends with TTL 64. It arrived at 62, so the packet was **routed twice** — once at the ingress VTEP and once at the egress VTEP. That is the definition of symmetric.

| Path | Routing hops | TTL at destination |
|---|---|---|
| Bridged, same subnet | 0 | unchanged |
| Asymmetric IRB | 1 (ingress only) | 63 |
| Symmetric IRB | 2 (ingress + egress) | **62** |

Spines never affect this count. They route the *outer* VXLAN header, so the inner packet's TTL passes through untouched no matter how many spine hops the fabric has. The same-subnet case is a useful control: a ping across the stretched VLAN 20 arrives with its TTL completely unchanged, because that path is pure bridging.

**2. Look for the L3 VNI.** The structural answer — if a VRF-to-VNI binding exists and traffic uses it, you are symmetric by construction:

```text
show vxlan vni
show interfaces Vxlan1
```

Two lines matter in `show interfaces Vxlan1` — the static VRF mapping you configured, and the internal VLAN EOS allocated to carry it:

```text
Static VRF to VNI mapping is [TENANT_A, 50000]
Dynamic VLAN to VNI mapping for 'evpn' is [4092, 50000]
```

An asymmetric fabric has neither line.

**3. Check for Type-5 routes.** Symmetric IRB advertises tenant subnets as EVPN **Type-5 (IP Prefix)** routes carrying the L3 VNI and a Router MAC extended community. Asymmetric IRB produces no Type-5 routes for tenant subnets at all — it works purely from Type-2, because the ingress VTEP needs the remote *host's* MAC and ARP binding rather than a prefix:

```text
show bgp evpn route-type ip-prefix
show ip route vrf TENANT_A
```

The distinction shows up inside the Type-2 routes too. Arista's IRB guide notes that "the symmetric IRB type-2 route contains a number of additional extended community attributes over the asymmetric IRB type-2 route" — the additions being the second label (the L3 VNI) and the Router MAC:

```text
show bgp evpn route-type mac-ip 0050.7966.6807 detail
```

**4. Read the VNI off the wire.** The unambiguous data-plane proof. Capture on a spine-to-leaf link in EVE-NG while pinging between VLANs, and look at the VNI field in the VXLAN header:

- **VNI 50000** — the L3 VNI — means symmetric.
- **VNI 1020** — the destination VLAN's L2 VNI — means asymmetric.

Same-subnet traffic shows the L2 VNI either way, so only the inter-subnet flow distinguishes them.

**5. Delete the destination SVI.** Proof by construction, if you want it settled beyond argument. Remove `interface Vlan10` from Leaf3 and re-run a ping from VPC2 to a VLAN 10 host.

Asymmetric IRB **cannot** survive this — the ingress VTEP must route into VLAN 10 locally before bridging, so with no `Vlan10` there is no path. Symmetric IRB does not care: Leaf3 routes into the IP-VRF and lets the far leaf handle the last hop. If the ping still succeeds, you are definitively symmetric. This is not a lab trick either — it is precisely the property that lets a leaf carry only the subnets it actually hosts.

Summary of what each check shows:

| Check | Asymmetric | Symmetric |
|---|---|---|
| TTL at destination | 63 | 62 |
| `show vxlan vni` | L2 VNIs only | L2 VNIs + VRF VNI |
| `show interfaces Vxlan1` | no VRF mapping | `Static VRF to VNI mapping` present |
| EVPN Type-5 routes | none | present, with L3 VNI + Router MAC |
| VNI on the wire, inter-subnet | destination L2 VNI | L3 VNI |
| Survives deleting the destination SVI | no | yes |

### 5.8 Which model should you use

**Symmetric IRB, essentially always.** In production fabrics it is close to universal — Arista's validated designs and ATD lab guides are built on it, and it is what you will meet in the field. Asymmetric IRB survives mainly in small single-tenant deployments and in teaching material.

The reason is the requirement quoted back in 5.1: asymmetric needs every participating VTEP to be a member of **all** the tenant's subnets, with an anycast SVI for each, and to hold every tenant MAC and ARP binding. That is fine for the three VLANs and three leaves here. It becomes both an operational burden and a hard scaling ceiling on MAC and ARP table size the moment leaves stop hosting every VLAN — which is immediately, in any fabric with more than a handful of tenants.

So a practical recommendation for working through this post: **build section 5.6 and skip building the asymmetric variant.** Sections 5.2 to 5.5 are worth reading, because the mechanics they explain — anycast gateways, autostate, the MLAG shared router MAC, and why a flood-and-learn fabric needs a virtual VTEP — all still apply underneath the symmetric design. But configuring asymmetric IRB end to end teaches you a model you are unlikely to deploy, and the lab time is better spent on the one you will. If you want to *see* the difference rather than build it, the TTL check in 5.7 shows it in a single ping.

The one case that still justifies asymmetric is a genuinely small, single-tenant fabric where every leaf really does host every VLAN and adding a tenant VRF buys you nothing. Even then, symmetric costs only a handful of extra lines and leaves you somewhere better when the fabric grows.

One honest caveat about reproducing section 5.6 in EVE-NG: VXLAN routing on vEOS-lab is not something Arista publishes a support matrix for. Community labs and the netlab project's EVPN platform matrix report both asymmetric and symmetric IRB working on virtual EOS images, and unlike the multicast data plane in section 3 there is no known blocker. Some `show vxlan config-sanity` platform-dependent rows may report VXLAN routing as "not enabled" on a software image even when it is forwarding correctly, so trust the ping and the route table over that row. On real hardware there are platform prerequisites the virtual image does not have — R and R2 series need `hardware tcam profile vxlan-routing`, and Trident2 and some Tomahawk platforms need `channel-group recirculation` — so check the VXLAN configuration guide for your exact model before deploying this.

## 6. Summary: HER vs multicast, IGMP snooping, versions, and queriers

### 6.1 HER vs multicast underlay

Both solve the same problem — delivering one BUM frame to N remote VTEPs — at opposite ends of a classic trade: **HER spends bandwidth to keep the network stateless; multicast spends state and protocol complexity to keep bandwidth minimal.**

With HER, the ingress VTEP makes N-1 unicast copies. A broadcast entering a fabric of 100 VTEPs leaves the ingress leaf's uplinks 99 times. With a multicast underlay it leaves once, and the PIM tree forks it only where paths actually diverge — spines replicate, links never carry duplicates.

| | Head-end replication | Multicast underlay |
|---|---|---|
| Copies on ingress uplink | one per remote VTEP in the VNI | exactly one |
| Underlay requirements | plain IP unicast | PIM-SM everywhere, RP design, RPF-clean topology |
| State in the core | none — spines just route unicast | (*,G)/(S,G) per group on every router in the tree |
| Flood list / tree maintenance | static: manual (section 2); EVPN: automatic Type-3 (section 4) | PIM joins/prunes; EVPN PMSI signals group membership (section 4.1) |
| Troubleshooting | easy — it's unicast; ping and traceroute tell the truth | mroute state, RPF failures, RP health all in play |
| Failure domain | per-VTEP | RP is critical shared infrastructure (mitigate with anycast RP) |
| Scales badly when | many VTEPs per VNI **and** lots of BUM (each of N VTEPs replicating to N-1 peers is O(N²) fabric-wide) | operational scale: many groups, many trees, multicast expertise required |
| Virtual lab support | works on vEOS-lab | data plane unsupported on vEOS-lab (sections 3, 4.1) |
| Typical fit | the default; small-to-large fabrics with modest BUM | very large fabrics, or overlays carrying real multicast applications (market data, IPTV, storage sync) |

Rules of thumb:

- **Default to EVPN + HER.** ARP suppression removes most broadcast before replication even matters, and a fabric with no multicast in the underlay is dramatically simpler to operate. This is also Arista's mainstream deployment model.
- Move to a multicast underlay when BUM volume is genuinely high — usually because tenants **run multicast applications over the overlay** — or when VTEP counts per VNI reach the point where ingress replication measurably loads leaf uplinks.
- Group-to-VNI mapping is a design knob: one group per VNI (as here) gives precise delivery but more state; sharing one group across many VNIs shrinks state but delivers BUM to VTEPs that must then drop it — the same bandwidth-vs-state trade-off one level down.

### 6.2 IGMP snooping

Everything above moves multicast **between** routers. IGMP snooping is the L2 half of the story: inside a VLAN, a switch treats multicast like broadcast and floods it to every port unless something tells it who the receivers are. A snooping switch listens to the IGMP conversation between hosts and routers, learns which ports have members of which group, and constrains forwarding to member ports plus multicast-router (mrouter) ports.

Where it matters in this lab's context:

- On **host-facing VLANs**: if VPC2's VLAN 20 carried a 10 Gb/s multicast feed, without snooping every port in VLAN 20 on that switch receives it, interested or not.
- On the **underlay** it's irrelevant here — all fabric links are routed /31s with no L2 segment to snoop.
- In the **overlay**, snooping composes with EVPN: EOS supports IGMP snooping over VXLAN, and EVPN Type-6 (SMET) routes extend it fabric-wide, so a VTEP with no interested receivers doesn't get overlay multicast at all — snooping's port-level idea lifted to the VTEP level.

One classic trap: a snooping switch only learns from IGMP packets it actually sees, and hosts only send reports when queried. Which is why versions and queriers matter.

### 6.3 IGMPv1 vs v2 vs v3

| | IGMPv1 (RFC 1112) | IGMPv2 (RFC 2236) | IGMPv3 (RFC 3376) |
|---|---|---|---|
| Join | Membership Report | Membership Report | Report to 224.0.0.22, with source lists |
| Leave | **silent** — stop reporting, group times out (minutes) | Leave Group to 224.0.0.2; router sends Group-Specific Query | per-source leave via INCLUDE/EXCLUDE state change |
| Querier election | none in the protocol (left to the routing protocol/DR) | lowest IP address wins | lowest IP address wins |
| Source selection | any source | any source | **source filtering** — "group G only from source S" |
| Enables | ASM only | ASM, fast leave | **SSM (232.0.0.0/8)** — no RP needed, host names the source |
| Report suppression | yes | yes (one host answers per group) | **no** — every host reports, so snooping switches see every member |

Practical notes:

- v2's Leave + Group-Specific Query cut leave latency from minutes to seconds — that alone made v1 obsolete.
- v3's per-host unsuppressed reports are a gift to IGMP snooping: the switch sees every member on every port instead of one spokesman per group.
- Version mismatches degrade to the lowest common version on the segment, losing v3's source filtering — pin versions where SSM matters.
- SSM (v3-only) removes the RP entirely: the host asks for (S,G) directly and the tree is built straight to the source. If this lab's underlay used SSM-mapped groups, Border-as-RP would not exist as a failure point at all.

### 6.4 When you need an IGMP querier

Snooping is a **listener** — it depends on somebody transmitting periodic General Queries to make hosts refresh their membership. Normally the PIM router on the VLAN is that somebody (in this lab's multicast sections, the leaves would be).

The failure mode appears on **L2-only VLANs with snooping enabled and no multicast router**: nobody queries, hosts never re-report, the snooping entries age out (~2-3 minutes), and multicast either goes dark or collapses back to flooding depending on the platform. It's a notorious "multicast worked for two minutes then died" symptom.

The fix is a snooping querier — a switch faking the router's query role:

```text
ip igmp snooping querier
ip igmp snooping vlan 20 querier
ip igmp snooping vlan 20 querier address 192.168.20.1
```

Guidelines: enable a querier on every snooped VLAN that has no PIM router; give it a source address valid for that subnet (a low one, since lowest-IP wins election if a real router shows up later — the real querier should win); one active querier per VLAN with a second switch configured as standby is plenty.

### 6.5 Closing checklist

The one-paragraph version of this whole lab: **build a boring OSPF underlay, let EVPN build the flood lists (HER) and suppress the ARP that would have needed them, keep MLAG peers presenting one shared VTEP IP, remember that bridging across the fabric and routing across the fabric are two separate features, and reach for a multicast underlay only when BUM volume or overlay multicast applications justify carrying PIM state in the core — and when that day comes, test it on hardware, because vEOS-lab will happily run the entire multicast control plane while forwarding none of it.**

The four failures that cost the most time here, and the command that catches each one:

| Symptom | Cause | Command that finds it |
|---|---|---|
| No BUM crosses the fabric; remote MACs never learned | VXLAN source interface does not match the IP remote VTEPs are addressing | `show interfaces vxlan 1 \| include Source` |
| PIM up, RP known, `show ip mroute` permanently empty | vEOS-lab does not implement the VXLAN multicast data plane | `show ip mroute` on every node |
| Same-VLAN pings work, inter-VLAN pings fail | SVI down from autostate, or missing virtual VTEP IP | `show ip interface brief`, `show vxlan config-sanity detail` |
| Routed traffic hairpins across the MLAG peer-link | MLAG shared router MAC not enabled | `show interfaces vxlan 1 \| include Shared Router MAC` |

`show vxlan config-sanity detail` is worth running after every change in this lab. It catches three of the four on its own.

And the final state of the lab — `show vxlan config-sanity detail` on all three leaves with everything in this post applied, every check green, including the `Routing` row that failed back in section 5.3:

```text

Leaf1#show vxlan config-sanity detail
Category                            Result 
---------------------------------- --------
Local VTEP Configuration Check        OK   
  Loopback IP Address                 OK   
  VLAN-VNI Map                        OK   
  Flood List                          OK   
  Routing                             OK   
  VNI VRF ACL                         OK   
  Decap VRF-VNI Map                   OK   
  VRF-VNI Dynamic VLAN                OK   
Remote VTEP Configuration Check       OK   
  Remote VTEP                         OK   
Platform Dependent Check              OK   
  VXLAN Bridging                      OK   
  VXLAN Routing                       OK   
CVX Configuration Check               OK   
  CVX Server                          OK   
MLAG Configuration Check              OK   
  Peer VTEP IP                        OK   
  MLAG VTEP IP                        OK   
  Peer VLAN-VNI                       OK   
  Virtual VTEP IP                     OK   
  MLAG Inactive State                 OK   

Detail                                            
--------------------------------------------------
                                        
Not in controller client mode                     
Run 'show mlag config-sanity' to verify MLAG config

Leaf1#

Leaf2#show vxlan config-sanity detail
Category                            Result 
---------------------------------- --------
Local VTEP Configuration Check        OK   
  Loopback IP Address                 OK   
  VLAN-VNI Map                        OK   
  Flood List                          OK   
  Routing                             OK   
  VNI VRF ACL                         OK   
  Decap VRF-VNI Map                   OK   
  VRF-VNI Dynamic VLAN                OK   
Remote VTEP Configuration Check       OK   
  Remote VTEP                         OK   
Platform Dependent Check              OK   
  VXLAN Bridging                      OK   
  VXLAN Routing                       OK   
CVX Configuration Check               OK   
  CVX Server                          OK   
MLAG Configuration Check              OK   
  Peer VTEP IP                        OK   
  MLAG VTEP IP                        OK   
  Peer VLAN-VNI                       OK   
  Virtual VTEP IP                     OK   
  MLAG Inactive State                 OK   

Detail                                            
--------------------------------------------------
        
Not in controller client mode                     
Run 'show mlag config-sanity' to verify MLAG config
                      
Check this command from the peer                  
                                                  
Leaf2#                                         


Leaf3#show vxlan config-sanity detail
Category                            Result 
---------------------------------- --------
Local VTEP Configuration Check        OK   
  Loopback IP Address                 OK   
  VLAN-VNI Map                        OK   
  Flood List                          OK   
  Routing                             OK   
  VNI VRF ACL                         OK   
  Decap VRF-VNI Map                   OK   
  VRF-VNI Dynamic VLAN                OK   
Remote VTEP Configuration Check       OK   
  Remote VTEP                         OK   
Platform Dependent Check              OK   
  VXLAN Bridging                      OK   
  VXLAN Routing                       OK   
CVX Configuration Check               OK   
  CVX Server                          OK   
MLAG Configuration Check              OK   
  Peer VTEP IP                        OK   
  MLAG VTEP IP                        OK   
  Peer VLAN-VNI                       OK   
  Virtual VTEP IP                     OK   
  MLAG Inactive State                 OK   

Detail                                            
--------------------------------------------------
    
Not in controller client mode                     
Run 'show mlag config-sanity' to verify MLAG config
MLAG peer is not connected                        
                                                  
Leaf3#
```
