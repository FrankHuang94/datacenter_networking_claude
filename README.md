# Datacenter Networking Technology Knowledge Database

A structured, in-depth knowledge base on datacenter networking, spanning the full interconnect hierarchy — from chip-level die-to-die interconnects (UCIe, HBM), through board-level fabrics (PCIe, CXL), to rack- and pod-scale networks (Ethernet, InfiniBand, RDMA), the optical layer (coherent optics, ROADMs, DCI, submarine, co-packaged optics), the switching silicon and AI fabrics, and the vendor landscape, future roadmaps, and sustainability constraints that bound the whole field.

## Executive Summary

Datacenter networking has been transformed from supporting plumbing into a first-order determinant of performance and economics, driven above all by the rise of AI and HPC workloads whose synchronized, all-to-all, latency-sensitive collective communication demands lossless, ultra-low-latency, ultra-high-bandwidth fabrics. This database treats the entire interconnect hierarchy as a single subject, because in the AI era it has effectively become one: a single training step touches every layer, from HBM-attached GPU memory, across NVLink within a node, across InfiniBand or RoCE between nodes, over optical DCI between regions. Understanding any one layer in isolation is no longer sufficient.

The database is organized as a progression from the inside out and from the physical to the strategic. It begins with the strategic context and the protocol stack, descends to the chip-level interconnects (PCIe, CXL, UCIe), rises through the rack-level fabrics (Ethernet, InfiniBand, RDMA), traverses the optical layer in depth (fundamentals, coherent, switching, DCI, co-packaged optics), examines the switching silicon and the AI fabric architectures that integrate everything, and concludes with the operational disciplines (network OS, lossless-fabric deployment, storage, security, virtualization, telemetry), the transceiver market, the vendor landscape, the future roadmaps, and the power and sustainability constraints.

Throughout, a set of structural tensions recurs: Ethernet versus InfiniBand for AI; electrical versus optical interconnect; merchant silicon versus custom ASIC; and proprietary versus open standards (NVLink versus UALink, InfiniBand versus Ethernet, integrated optical systems versus disaggregated open line systems). These tensions are the competitive and strategic expression of the underlying technology trajectories, and tracing them is one of the database's organizing themes. Several of them have partly resolved since this database was first written — Ethernet passed InfiniBand in AI back-end networks, co-packaged optics reached production, and the open scale-out and scale-up specifications were all published — and the resolutions are more interesting than either camp predicted, because winning the protocol argument turned out not to be the same as winning the market. The August 2026 update section at the end of this README summarizes what moved.

The intended audience spans engineers, architects, product and strategy professionals, and technically inclined investors who need a coherent, in-depth, cross-referenced reference on datacenter networking technology — one that connects the physics of a modulator to the economics of a hyperscaler's fabric, and the architecture of a switch ASIC to the strategy of the companies that build it.

## Table of Contents

1. **[Overview and Strategic Context](01_overview_and_strategic_context.md)** — What datacenter networking encompasses, the AI inflection, the scale of hyperscale networks, the interconnect hierarchy, standards bodies, market size, and the key technology tensions.
2. **[OSI Model and Datacenter Protocol Stack](02_osi_model_and_datacenter_protocol_stack.md)** — The protocol stack layer by layer, with datacenter-specific implementations, and how the layers co-design into a lossless AI fabric; overlays (VXLAN, GENEVE).
3. **[PCIe Deep Dive](03_pcie_deep_dive.md)** ⭐ — PCIe architecture, generations 1–8, the three-layer protocol stack, power management, datacenter roles and bottlenecks, and the ecosystem.
4. **[CXL Deep Dive](04_cxl_deep_dive.md)** ⭐ — Compute Express Link: origins, the three sub-protocols, device types, memory pooling and switching, fabric/peer-to-peer (3.0/3.1), the 4.0 generation, the competitive landscape, and the software stack.
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
19. **[Network Security and Zero Trust](19_network_security_and_zero_trust.md)** — Zero trust, micro-segmentation, DDoS, MACsec/IPsec, post-quantum cryptography, DPU security offload, BGP security, and supply-chain security.
20. **[Network Virtualization and Overlay](20_network_virtualization_and_overlay.md)** — VXLAN/EVPN, GENEVE, SRv6/MPLS, NSX, eBPF/XDP, and Kubernetes CNI.
21. **[400G/800G/1.6T Transceiver Market](21_400g_800g_1pt6t_transceiver_market.md)** — Market sizing and growth, the transceiver market by speed generation through 3.2T/448G, the supplier landscape, and the technology trends.
22. **[Network Telemetry and Observability](22_network_telemetry_and_observability.md)** — INT, streaming telemetry, sFlow/IPFIX, BMP, AIOps, and eBPF observability.
23. **[Vendor Landscape and Competitive Analysis](23_vendor_landscape_and_competitive_analysis.md)** ⭐ — Deep profiles of Cisco, Arista, Juniper (now HPE Networking), NVIDIA, Broadcom, Marvell, Ciena, Nokia (including the acquired Infinera), Lumentum, and Coherent, with market-share summaries and the 2025 consolidation wave.
24. **[Future Roadmaps and Emerging Technologies](24_future_roadmaps_and_emerging_technologies.md)** ⭐ — What has already landed as of 2026, electrical scaling to 448G, next-gen coherent, switch-ASIC roadmaps, the 100K-GPU challenge, UALink and scale-up Ethernet (ESUN/SUE), scale-across, photonic computing, quantum, 6G, and memory-semantic networking.
25. **[Sustainability, Power, and Cooling](25_sustainability_power_and_cooling.md)** — Networking's power share, AI escalation, CPO/LPO savings, liquid cooling, 800 VDC power delivery and the 1 MW rack, carbon, and efficiency metrics.
26. **[Glossary](26_glossary.md)** — Cross-referenced acronyms and terms.

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

This database synthesizes publicly available technical knowledge on datacenter networking as of its last update, organized into a coherent, cross-referenced reference. It emphasizes technical depth and roadmap detail, and it favors full prose with tables and worked explanations over bullet outlines. Specific figures (speeds, dates, market shares, product details) reflect the state of the field as of the last update and the published roadmaps current at that time; readers should verify rapidly changing specifics (particularly product roadmaps and market shares) against primary sources.

**Last updated:** 2026-08-13

### What Changed in the August 2026 Update

The refresh covered the roadmap, market, and product facts that had moved since the previous edition, and added several topics the earlier writing did not cover at all.

**Corrections to previously projected items now confirmed or superseded:**

| Item | Previous treatment | Current status |
|---|---|---|
| Ultra Ethernet | Consortium effort in progress | **Spec 1.0 published June 2025** (~560 pages); 1.0.2 revision 2026; implementations shipping (Files 06, 17, 24) |
| UALink | "UALink 1.0 (2024), ~200 GB/s per link" | **UALink 200G 1.0, April 2025**: 200 Gbps *per lane*, 1,024 accelerators; switch silicon late 2026 (Files 15, 24) |
| CXL 4.0 | Projected ~2027+ | **Released November 2025** on PCIe 7.0; bundled ports to ~1.5 TB/s (Files 04, 24) |
| PCIe 7.0 | "Targeted ~2025" | **Released June 2025**; PCIe 8.0 draft to members February 2026 (File 03) |
| Co-packaged optics | Pilot/proof-of-concept in 2025–2026 | **In production** — Broadcom TH6-Davisson, NVIDIA Quantum-X and Spectrum-X Photonics (Files 13, 14, 21) |
| 102.4T switch silicon | Roadmap item | **Shipping in volume since 2025** (Files 14, 24) |
| HBM4 | "~2026 target, ~2 TB/s" | **Mass production 2026**; 2048-bit interface, logic base die, ~48 GB/stack (File 05) |
| Ethernet vs. InfiniBand | Contest in progress | **Ethernet passed InfiniBand in 2025**, ~two-thirds of AI-cluster switch revenue — but much of it is NVIDIA's Spectrum-X (Files 07, 23, 24) |
| HPE–Juniper | "Announced 2024, pending" | **Closed July 2025** after DOJ settlement (File 23) |
| Nokia–Infinera | "Subject of acquisition interest" | **Closed February 2025**; Nokia now #2 in optical at ~20% (Files 10, 11, 12, 23) |
| NVIDIA networking revenue | "Several billion dollars per year" | **~$31B in fiscal 2026**; Spectrum-X past $10B annualized (Files 01, 14, 23) |

**Topics added that were previously absent:**

- **Scale-up Ethernet** — the OCP **ESUN** workstream, Broadcom's **SUE**, and NVLink Fusion, opening a third standards front alongside scale-out and the optical layer (Files 06, 24).
- **Scale-across** — joining geographically separated AI clusters into one training domain, driven by per-site power limits; Cisco P200/8223 and NVIDIA Spectrum-XGS (Files 12, 14, 24).
- **Post-quantum cryptography** — the "harvest now, decrypt later" threat model, NIST's ML-KEM/ML-DSA/SLH-DSA, hybrid key exchange, and crypto-agility as a procurement requirement for long-lived network hardware (File 19).
- **800 VDC power delivery and the 1 MW rack** — why megawatt racks force DC distribution, and what that means for liquid-cooled switches and co-packaged optics (File 25).
- **448G per lane and 3.2T optics** — the generation after 1.6T, demonstrated at OFC 2026, plus the LPO/LRO/NPO/CPO continuum (Files 21, 24).
- **Vera Rubin, AMD Helios, and TPU v7 Ironwood** as the current reference architectures (Files 07, 14, 15).
- **Optical transceiver market sizing** and the **InP laser supply constraint** now bounding the bandwidth roadmap (Files 01, 21).
- **The 2026 subsea buildout** — Meta's Waterworth, AWS Fastnet, geopolitical route selection, and the FCC's 2026 licensing overhaul (File 12).

**Future additions under consideration:** an expanded standalone index, deeper worked examples and case studies, and periodic refresh of the roadmap and market-share data as the field evolves.
