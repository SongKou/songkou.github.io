+++
title = 'Who I Am'
date = 2026-10-03T12:00:00+08:00
draft = false
# Pinned: a non-zero weight sorts this post ahead of every dated post, so it
# stays first on the home page. Delete the line to unpin it.
weight = 1
description = 'About me'
categories = ['About']
tags = ['About']
+++

I'm Song Kou, a network engineer in Singapore. For more than 17 years I have designed, built and operated large data center networks, and I write the code that deploys them. I'm currently a Principal Network Engineer at Equinix, where I own the design and implementation of the Equinix Fabric network across Asia-Pacific.

This blog is my notebook. The oldest notes date from 2020, when it was a place to keep commands I kept forgetting. It has since grown into long lab write-ups on VXLAN EVPN, multicast, RoCEv2 and the networking underneath AI clusters. This page stays pinned at the top to say who is writing and where to start reading.

## What I work on

- **Data center fabrics.** MP-BGP EVPN and VXLAN, leaf-spine design, IP storage networks and Cisco ACI Multipod. I have also run the designs that came before them in production: OTV, FabricPath, vPC and Fabric Extenders.
- **Routing and multicast.** BGP, OSPF, IGMP, PIM Sparse Mode, Anycast RP and MSDP.
- **Automation.** Python, NETCONF through ncclient, Jinja2 templates and Git.
- **Platforms.** Cisco NX-OS, ACI and IOS, Arista EOS, Juniper Junos, and white-box platforms.
- **Certifications.** CCIE Routing and Switching, earned in 2011, and CCIE Data Center, earned in 2019. I also passed the Certified Kubernetes Administrator exam in 2022 and held AWS Certified Solutions Architect – Professional from 2022 to 2025.

## How I got here

**2006 to 2009: application support.** I graduated from Northwest University in China with a degree in Computer Science and Technology, and started in application support in Dalian. At Genpact I supported Microsoft servers and clients. At HiSoft I maintained RHEL, Solaris and Windows application servers for Eli Lilly and Company's drug discovery research center, led the China support team and automated system tasks with shell scripts.

**2009 to 2017: Cisco.** I spent eight years as a Network Consulting Engineer. The first two went to supporting service provider networks in the Americas and Europe. After that I took part in network design and implementation for Agricultural Bank of China and China Construction Bank, and designed and implemented networks for casino resorts in Macau and Singapore. Other projects included a core network upgrade for Telekom Malaysia, wireless rollouts, and network automation for NTT East on Cisco's NFV stack. I passed CCIE Routing and Switching in 2011.

**2018 to 2020: DBS Bank and Sea Group.** At DBS Bank I planned, designed and implemented the network for a new data center, automated its setup, and supported the migration out of the old data center. CCIE Data Center followed in 2019. At Sea Group (Garena) I then designed and delivered automated leaf-spine networks across four data centers.

**2020 to now: Equinix.** I joined as a Senior Network Engineer and became Principal in March 2025. I own the design and implementation of Equinix Fabric across APAC. That covers capacity expansion, network rollout for new IBX data centers, the region's configuration, racking and cabling standards, and guiding the APAC team's projects. A good part of the work is automation. I write Python tools that automate network deployment, upgrades and monitoring.

## What I'm learning now

In 2026 I started studying HPC and AI networking on my own time: lossless Ethernet for RoCEv2 with PFC, ECN and DCQCN, RDMA benchmarking, NVIDIA Cumulus Linux and ConnectX, SONiC, 400G and 800G optics, and what GPU workloads ask of the network.

This part is independent lab and study work rather than production experience. The labs run on virtual switches in EVE-NG and on Soft-RoCE virtual machines, and the posts say which parts I tested and which parts I could only research.

## Where to start reading

**Data center fabrics.** [VXLAN EVPN Architecture](/posts/vxlan-evpn-architecture/) is the reference piece, from encapsulation and route types to multihoming, Multi-Site and firewall insertion. The [Arista VXLAN EVPN lab](/posts/arista-vxlan-bum-her-vs-multicast/) builds one fabric four ways, adds inter-VLAN routing, then migrates it live from iBGP to eBGP and on to BGP unnumbered. The [Cumulus Linux lab](/posts/vxlan-evpn-cumulus-5.4-lab-guide/) builds a smaller EVPN fabric with an MLAG pair and a distributed anycast gateway.

**Multicast.** [PIM Sparse Mode in Detail](/posts/pim-sparse-mode-detailed/) walks the control plane step by step and ends with Anycast RP and multicast ECMP. [PIM DR, Assert, and Multicast ECMP](/posts/pim-dr-assert-multicast-ecmp/) answers the questions that come up when two PIM routers share a receiver VLAN.

**Lossless Ethernet for RoCEv2.** Start with [RoCE QoS Concepts and Packet Examples](/posts/roce-qos-concepts-and-packet-examples/) for the vocabulary. Then pick a platform: [Cisco Nexus, Cumulus Linux and ConnectX end to end](/posts/rocev2-cisco-cumulus-connectx-end-to-end/), [Arista EOS](/posts/arista-eos-roce-config/), or [Cumulus Linux in depth](/posts/roce_cumulus_linux/). [Perftest](/posts/perftest/) covers benchmarking, and [RDMA Performance Tuning](/posts/rdma-performance-tuning/) covers what to check when the bandwidth will not fill up.

**The GPU side.** [TP, PP, DP](/posts/tp-pp-dp-llm-parallelism/) explains how one model is split across several GPUs. [NCCL and NVLink Notes](/posts/nvlink-nccl-scaleup-scaleout/) covers what moves between them. [Who Chooses What?](/posts/nccl-workflow/) corrects a wrong mental model I had formed from the first two.

**Optics.** [SFP, QSFP, Fiber Types](/posts/sfp-qsfp-fiber-400g-800g-optics/) goes from ordinary SFP modules and fiber types to 400G and 800G.

**Automation and lab tooling.** [Configuration Standardization and Automation on Cumulus and SONiC](/posts/cumulus-sonic-config-cli/) documents a Python tool I wrote that renders one YAML file per device into vendor-native configuration, with diff and commit. There is also a [SaltStack guide](/posts/salt-guide/), a walkthrough for [SONiC on EVE-NG](/posts/eve-ng-6-sonic-virtual-switch-installation/), and a [cheat sheet](/posts/cheat-sheet/) that collects the verification commands from the lab posts.

**Fundamentals and older notes.** [How Linux Sends and Receives Network Packets](/posts/how_linux_sends_and_receives_network_packets/), [TCP handshake and termination](/posts/tcp_three_way_handshake_and_four_way_termination/), and [Low Latency Network Architecture](/posts/low_latency_network_architecture/) for trading networks. Earlier notes cover CDN, load balancing, Linux namespaces, MySQL, Python and Git. The [Categories](/categories/) and [Tags](/tags/) pages list everything.

## How the posts are written

- **Configs and output stay complete.** I come back to these posts to rebuild things, so configurations and command output are kept whole instead of trimmed to the interesting lines.
- **Tested or researched, and labeled.** Lab posts show real captures, including the mistakes. When a virtual lab cannot do something, or I could not test a claim on real hardware, the post says so.
- **Sources are linked.** Posts that started as study notes say so, and most of the recent ones end with a reference list.

## Contact

You can find me on [LinkedIn](https://www.linkedin.com/in/song-kou-62715210).

This is a personal blog. The labs are independent work and the views are my own, not my employer's.
