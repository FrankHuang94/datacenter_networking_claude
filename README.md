# Datacenter Networking Technology Knowledge Database

A structured, in-depth knowledge base on datacenter networking, spanning the full interconnect hierarchy — from chip-level die-to-die interconnects (UCIe, HBM), through board-level fabrics (PCIe, CXL), to rack- and pod-scale networks (Ethernet, InfiniBand, RDMA), the optical layer (coherent optics, ROADMs, DCI, submarine, co-packaged optics), the switching silicon and AI fabrics, and the vendor landscape, future roadmaps, and sustainability constraints that bound the whole field.

## Executive Summary

Datacenter networking has been transformed from supporting plumbing into a first-order determinant of performance and economics, driven above all by the rise of AI and HPC workloads whose synchronized, all-to-all, latency-sensitive collective communication demands lossless, ultra-low-latency, ultra-high-bandwidth fabrics. This database treats the entire interconnect hierarchy as a single subject, because in the AI era it has effectively become one: a single training step touches every layer, from HBM-attached GPU memory, across NVLink within a node, across InfiniBand or RoCE between nodes, over optical DCI between regions. Understanding any one layer in isolation is no longer sufficient.

The database is organized as a progression from the inside out and from the physical to the strategic. It begins with the strategic context and the protocol stack, descends to the chip-level interconnects (PCIe, CXL, UCIe), rises through the rack-level fabrics (Ethernet, InfiniBand, RDMA), traverses the optical layer in depth (fundamentals, coherent, switching, DCI, co-packaged optics), examines the switching silicon and the AI fabric architectures that integrate everything, and concludes with the operational disciplines (network OS, lossless-fabric deployment, storage, security, virtualization, telemetry), the transceiver market, the vendor landscape, the future roadmaps, and the power and sustainability constraints.

Throughout, a set of structural tensions recurs: Ethernet versus InfiniBand for AI; electrical versus optical interconnect; merchant silicon versus custom ASIC; and proprietary versus open standards (NVLink versus UALink, InfiniBand versus Ethernet, integrated optical systems versus disaggregated open line systems). These tensions are the competitive and strategic expression of the underlying technology trajectories, and tracing them is one of the database's organizing themes.

The intended audience spans engineers, architects, product and strategy professionals, and technically inclined investors who need a coherent, in-depth, cross-referenced reference on datacenter networking technology — one that connects the physics of a modulator to the economics of a hyperscaler's fabric, and the architecture of a switch ASIC to the strategy of the companies that build it.

## Table of Contents

1. **[Overview and Strategic Context](01_overview_and_strategic_context.md)** — What datacenter networking encompasses, the AI inflection, the scale of hyperscale networks, the interconnect hierarchy, standards bodies, market size, and the key technology tensions.
2. **[OSI Model and Datacenter Protocol Stack](02_osi_model_and_datacenter_protocol_stack.md)** — The protocol stack layer by layer, with datacenter-specific implementations, and how the layers co-design into a lossless AI fabric; overlays (VXLAN, GENEVE).
3. **[PCIe Deep Dive](03_pcie_deep_dive.md)** ⭐ — PCIe architecture, generations 1–7, the three-layer protocol stack, power management, datacenter roles and bottlenecks, and the ecosystem.
4. **[CXL Deep Dive](04_cxl_deep_dive.md)** ⭐ — Compute Express Link: origins, the three sub-protocols, device types, memory pooling and switching, fabric/peer-to-peer (3.0/3.1), the competitive landscape, and the software stack.
5. **[UCIe and Die-to-Die Interconnects](05_ucie_and_die_to_die_interconnects.md)** ⭐ — Chiplets and the need for die-to-die interconnect, UCIe, the proprietary fabrics (EMIB, Foveros, Infinity Fabric, NVLink-C2C, CoWoS, SoIC), and HBM.
6. **[Ethernet Fundamentals and Datacenter](06_ethernet_fundamentals_and_datacenter.md)** ⭐ — Ethernet history, the full speed-standard family (100G–1.6T), the physical layer, datacenter topologies, and congestion management (PFC, DCQCN, HPCC, Swift, Ultra Ethernet).
7. **[InfiniBand Deep Dive](07_infiniband_deep_dive.md)** ⭐ — InfiniBand history and the Mellanox acquisition, generations, architecture, the NVIDIA AI fabric products, and the InfiniBand-versus-Ethernet comparison.
8. **[RDMA and RoCE](08_rdma_and_roce.md)** — RDMA fundamentals, RoCE/iWARP/Soft-RoCE, the programming model, MPI and collectives, NVMe-oF, and RoCEv2 deployment.
9. **[Optical Networking Fundamentals](09_optical_networking_fundamentals.md)** ⭐ — The physics of light in fiber, fiber types, amplifiers, modulators, receivers, and WDM.
10. **[Coherent Optical and Transceiver Ecosystem](10_coherent_optical_and_transceiver_ecosystem.md)** ⭐ — Coherent DSP, modulation formats, FEC, form factors and MSAs, ZR/ZR+, DSP vendors, and open optical networking.
11. **[Optical Switching and ROADM](11_optical_switching_and_roadm.md)** — ROADM, WSS, OXC, OCS, silicon-photonic switches, and vendors.
12. **[DCI, Metro, Long-Haul, and Submarine](12_datacenter_interconnect_and_subsea.md)** — The DCI reach continuum, OTN, FlexE, metro/long-haul DWDM, and submarine cable systems.
13. **[Co-Packaged Optics and LPO](13_co_packaged_optics_and_lpo.md)** ⭐ — The bandwidth-power crisis, LPO, CPO architectures and optical engines, the laser challenge, products, and challenges.
14. **[Network Switching ASICs](14_network_switching_asics.md)** ⭐ — Switch-ASIC architecture (fabric, traffic management, pipeline, TCAM, SerDes) and the vendor portfolios.
15. **[AI Networking and HPC Fabric](15_ai_networking_and_hpc_fabric.md)** ⭐ — Training communication patterns, scale-up vs. scale-out, the NVIDIA/AMD/Intel/Google/Meta architectures, and collective algorithms.
16. **[Network OS, SDN, and Programmability](16_network_os_sdn_and_programmability.md)** — Traditional and open NOSes, SDN, intent-based networking, P4, gNMI/OpenConfig, and verification.
17. **[Lossless RoCEv2 Fabric Deployment](17_roce_and_lossless_fabric_deployment.md)** — The operational craft of designing and tuning lossless AI fabrics (PFC, ECN, DCQCN, ECMP), and InfiniBand operations.
18. **[Storage Networking](18_storage_networking.md)** — Fibre Channel, iSCSI, NVMe-oF, distributed storage, and composable disaggregated infrastructure.
19. **[Network Security and Zero Trust](19_network_security_and_zero_trust.md)** — Zero trust, micro-segmentation, DDoS, MACsec/IPsec, DPU security offload, BGP security, and supply-chain security.
20. **[Network Virtualization and Overlay](20_network_virtualization_and_overlay.md)** — VXLAN/EVPN, GENEVE, SRv6/MPLS, NSX, eBPF/XDP, and Kubernetes CNI.
21. **[400G/800G/1.6T Transceiver Market](21_400g_800g_1pt6t_transceiver_market.md)** — The transceiver market by speed generation, the supplier landscape, and the technology trends.
22. **[Network Telemetry and Observability](22_network_telemetry_and_observability.md)** — INT, streaming telemetry, sFlow/IPFIX, BMP, AIOps, and eBPF observability.
23. **[Vendor Landscape and Competitive Analysis](23_vendor_landscape_and_competitive_analysis.md)** ⭐ — Deep profiles of Cisco, Arista, Juniper, NVIDIA, Broadcom, Marvell, Ciena, Nokia, Infinera, Lumentum, and Coherent, with market-share summaries.
24. **[Future Roadmaps and Emerging Technologies](24_future_roadmaps_and_emerging_technologies.md)** ⭐ — Electrical scaling, next-gen coherent, switch-ASIC roadmaps, the 100K-GPU challenge, UALink, photonic computing, quantum, 6G, and memory-semantic networking.
25. **[Sustainability, Power, and Cooling](25_sustainability_power_and_cooling.md)** — Networking's power share, AI escalation, CPO/LPO savings, liquid cooling, power delivery, carbon, and efficiency metrics.
26. **[Glossary](26_glossary.md)** — Cross-referenced acronyms and terms.
27. **[Chinese Simplified Translation (简体中文翻译)](27_chinese_simplified_translation.md)** — A complete, precise Simplified Chinese translation of the entire database (README + Files 01–26), organized section by section, each linking back to its authoritative English source file.

(⭐ denotes a primary, maximum-depth chapter.)

## How to Navigate

**For newcomers**, a recommended reading order:
1. Start with **File 01** (strategic context) and **File 02** (the protocol stack) for the conceptual map.
2. Read **File 09** (optical fundamentals) for the physics background that much of the rest assumes.
3. Read **File 06** (Ethernet) and **File 07** (InfiniBand) for the two great rack-level fabric families, then **File 08** (RDMA) for the transport beneath them.
4. Read **File 15** (AI networking) as the synthesis that ties the fabrics together for the AI use case.
5. Branch into the areas of interest: chip-level (Files 03–05), optical depth (Files 10–13), silicon (File 14), operations (Files 16–22), and strategy/future (Files 23–25).

**For depth on a specific technology**, go directly to its chapter; each is cross-referenced to related chapters.

**For the competitive and strategic picture**, read Files 01, 15, 23, and 24 together.

## Methodology and Status

This database synthesizes publicly available technical knowledge on datacenter networking as of its last update, organized into a coherent, cross-referenced reference. It emphasizes technical depth and roadmap detail, and it favors full prose with tables and worked explanations over bullet outlines. Specific figures (speeds, dates, market shares, product details) reflect the state of the field in the mid-2020s and the published roadmaps current at that time; readers should verify rapidly changing specifics (particularly product roadmaps and market shares) against primary sources.

**Last updated:** 2026-08-27

**Future additions under consideration:** an expanded standalone index, a units/notation appendix, deeper worked examples and case studies, and periodic refresh of the roadmap and market-share data as the field evolves.
