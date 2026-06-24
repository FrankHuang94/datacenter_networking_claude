# Datacenter Networking — Overview, Scale, and Strategic Context

## What Datacenter Networking Encompasses

Datacenter networking is the discipline of moving information between computational elements at every scale of physical proximity, from transistors separated by micrometers on adjacent silicon dies to data halls separated by thousands of kilometers of submarine fiber. It is tempting to think of "the network" as the rack-mounted switches and the cables between racks, but that view is decades out of date. Modern datacenter networking is better understood as a continuous interconnect hierarchy in which the same fundamental engineering problem — move bits from point A to point B with the highest bandwidth, lowest latency, lowest energy per bit, and acceptable cost — recurs at every length scale, each with its own dominant physics, dominant protocols, and dominant vendors.

At the shortest length scale, **die-to-die interconnect** moves data between chiplets co-packaged on a single substrate. Here the medium is microscopic copper bumps or hybrid-bonded copper pads, the distances are sub-millimeter to a few millimeters, and the relevant standards are UCIe (Universal Chiplet Interconnect Express), HBM (High Bandwidth Memory) interfaces, and a constellation of proprietary fabrics such as Intel's EMIB and Foveros, AMD's Infinity Fabric, NVIDIA's NVLink-C2C, and TSMC's CoWoS and SoIC packaging. Energy budgets here are extraordinarily tight — under 1 picojoule per bit — and bandwidth densities reach tens of terabytes per second per millimeter of interface edge.

One step out, **chip-to-chip on-package and on-board interconnect** moves data between full packages on a printed circuit board or between sockets in a server. PCIe (Peripheral Component Interconnect Express) and its cache-coherent superset CXL (Compute Express Link) dominate the CPU-to-device path; NVLink and its switched fabric NVSwitch dominate the GPU-to-GPU path inside an accelerator node; and emerging open standards such as UALink (Ultra Accelerator Link) aim to break NVIDIA's proprietary lock on scale-up accelerator meshes.

At the **rack and pod scale**, two great rival families compete: Ethernet, the universal best-effort packet network that has conquered nearly every networking domain it has entered, and InfiniBand, the credit-based, lossless, RDMA-native fabric that has dominated high-performance computing and, until recently, AI training clusters. Between racks and across a datacenter, the network is almost universally a Clos (fat-tree) topology built from leaf and spine switches, increasingly supplemented by optical circuit switching for the largest AI fabrics.

At the **campus, metro, and long-haul scale**, the medium becomes optical fiber and the technology becomes coherent optical transmission, wavelength division multiplexing (WDM), reconfigurable optical add-drop multiplexers (ROADMs), and — for datacenter interconnect (DCI) — pluggable coherent optics such as 400G-ZR and 800G-ZR+. Finally, at the **planetary scale**, submarine cables with optical repeaters every 40–80 kilometers carry the inter-continental traffic that knits the global cloud together, with hyperscalers now owning entire private cable systems.

This database treats all of these as a single subject because, increasingly, they are. The AI workload that fine-tunes a large language model touches every one of these layers in a single training step: gradients computed in HBM-attached GPU memory, reduced across NVLink within a node, aggregated across InfiniBand or RoCE between nodes, checkpointed to disaggregated storage over NVMe-over-Fabrics, and — for geographically distributed training — synchronized across DCI links spanning metro and long-haul optical systems. Understanding any one layer in isolation is no longer sufficient.

## Why It Matters Now: The AI Networking Inflection

For roughly three decades, datacenter traffic was dominated by what network architects call **north-south** flows: a client somewhere on the internet makes a request, the request enters the datacenter through a load balancer, hits a web tier, which queries an application tier, which queries a database, and a response flows back out. The defining characteristic of north-south traffic is that it is relatively low in aggregate bandwidth per server, bursty, latency-tolerant at the millisecond scale, and — critically — tolerant of packet loss, because the TCP transport layer was designed precisely to recover gracefully from the occasional dropped packet.

The cloud era shifted the balance toward **east-west** traffic: server-to-server communication within the datacenter, driven by distributed databases, microservice meshes, MapReduce-style analytics, and distributed storage. East-west traffic grew faster than north-south for years, and it is the reason the industry abandoned the old three-tier access/aggregation/core topology (optimized for funneling traffic to and from the internet) in favor of the flat, high-bisection-bandwidth Clos fabric (optimized for any-server-to-any-server communication).

AI and HPC workloads have pushed this transformation to a qualitatively new regime. Distributed deep neural network training is characterized by **all-to-all** and **all-reduce** collective communication patterns in which, at the end of every training micro-step, every accelerator must exchange gradient or activation data with every other accelerator participating in the relevant parallelism dimension. The traffic is not merely east-west; it is **synchronized, bursty, bandwidth-enormous, and exquisitely latency-sensitive**. When 1,024 GPUs all finish a forward-backward pass at nearly the same instant and simultaneously launch an AllReduce, the network experiences a coordinated burst that can momentarily demand the full bisection bandwidth of the fabric. Any tail latency — a single slow link, a single congested switch buffer, a single dropped packet that triggers a Remote Direct Memory Access (RDMA) retransmission — stalls not just one flow but the entire collective, because the collective cannot complete until its slowest participant finishes. In a synchronous data-parallel training job, the GPUs sit idle waiting for the network, and idle GPUs at tens of thousands of dollars apiece are the single most expensive failure mode in the modern datacenter.

This is why three technologies that were once niche became existential for AI infrastructure:

- **RDMA (Remote Direct Memory Access)** moves data directly between the memory of two machines without involving either CPU, bypassing the kernel networking stack, eliminating copies, and reducing small-message latency from tens of microseconds (classic TCP sockets) to under a microsecond. RDMA is what lets a GPU's network interface deposit gradient data straight into a remote GPU's memory.

- **Lossless fabrics** ensure that the network never drops a packet due to congestion, because RDMA's reliable-connection transport recovers from loss far more expensively than TCP does — a single drop can force retransmission from the last acknowledged sequence number, devastating throughput. InfiniBand achieves losslessness natively through credit-based flow control; Ethernet achieves it through Priority Flow Control (PFC) plus congestion-control algorithms such as DCQCN.

- **Ultra-low and ultra-predictable latency** matters because collective operations are barriers: the tail of the latency distribution, not the mean, determines training throughput. A fabric with a good average but a long P99.9 tail is worse for synchronous training than a fabric with a slightly worse average but a tight tail.

The result is that networking, long treated as plumbing, has become a first-order determinant of AI cluster performance and economics. NVIDIA's 2020 acquisition of Mellanox for $6.9 billion was, in retrospect, one of the most strategically prescient moves in semiconductor history: it gave NVIDIA control of both the compute (GPU) and the interconnect (InfiniBand, ConnectX NICs, Quantum switches, BlueField DPUs) for AI training, and it is now a multi-billion-dollar revenue stream growing in lockstep with GPU demand.

## The Scale of Modern Datacenter Networks

The numbers involved in hyperscale networking are difficult to internalize. A single large hyperscaler operates on the order of **millions of switch ports** across its global fleet, interconnected by **tens of millions of optical transceivers** and **millions of kilometers of fiber**. Aggregate internal bandwidth within a single large datacenter building is measured in **petabits per second**; Google's published descriptions of its Jupiter datacenter fabric describe more than 6 petabits per second of bisection bandwidth in a single building generation, and subsequent generations have grown from there.

To make this concrete, consider the topology of a hyperscale Clos fabric. A modern leaf-spine pod might use 51.2 Terabit-per-second switch ASICs configured as 64 ports of 800G or 128 ports of 400G. A non-blocking two-tier Clos built from such switches, with a 1:1 oversubscription ratio appropriate for AI workloads, can interconnect thousands of servers within a pod at full bandwidth. Stacking pods into a three-tier Clos with a super-spine tier extends this to hundreds of thousands of endpoints. When the endpoints are GPUs rather than general-purpose servers — and when each GPU demands 400 to 800 gigabits per second of network bandwidth rather than the 25 to 100 gigabits per second typical of a general-compute server — the bandwidth requirements scale by another order of magnitude.

The hyperscalers each pursue this at planetary scale but with distinct architectural philosophies:

- **Google** pioneered the merchant-and-custom hybrid, building its own switch silicon and the Jupiter fabric, and was the first to deploy optical circuit switching (OCS) at scale in production datacenters, using MEMS-based optical switches to dynamically reconfigure topology. Google's TPU pods use a 3D torus interconnect with optical links, a radically different approach from the Clos fabrics used for general compute.

- **Meta** built much of its infrastructure on the Open Compute Project (OCP) hardware it helped found, deploying Broadcom-based switches (Wedge, Minipack) and large RoCEv2 Ethernet AI fabrics, while also operating substantial InfiniBand clusters for its largest GPU deployments. Meta has publicly described AI infrastructure on the order of 350,000 H100 GPUs.

- **Microsoft** open-sourced SONiC (Software for Open Networking in the Cloud), the Linux-based network operating system now used across much of the industry, and operates large NVIDIA-based AI fabrics (including Spectrum-X Ethernet) for the OpenAI partnership, alongside custom NICs (Azure Boost, MANA).

- **Amazon AWS** built the most aggressively custom infrastructure of the hyperscalers, including the Nitro system (offloading networking, storage, and security to custom silicon), its own Trainium and Inferentia AI accelerators with custom NeuronLink interconnect, and a proprietary network design philosophy that prioritizes blast-radius reduction and operational simplicity.

- **ByteDance** and other Chinese hyperscalers (Alibaba, Tencent, Baidu) operate at comparable scale, with Alibaba in particular publishing influential networking research (HPCC congestion control, high-precision in-band telemetry) and building large RoCE fabrics, while navigating US export controls on the most advanced GPUs and optical components.

## The Interconnect Hierarchy

It is worth laying out the full hierarchy explicitly, because the entire structure of this database follows it from the inside out:

| Tier | Scale | Dominant technologies | Energy/bit | Latency |
|---|---|---|---|---|
| Die-to-die (on-package) | µm–mm | UCIe, HBM, EMIB, Foveros, Infinity Fabric, NVLink-C2C, CoWoS, SoIC | <1 pJ/bit | <1 ns |
| Chip-to-chip (on-package) | mm–cm | NVLink, NVSwitch, UALink, Infinity Fabric | 1–2 pJ/bit | ~10–100 ns |
| Board-level | cm | PCIe, CXL | 3–5 pJ/bit | ~100 ns |
| Rack-level | m | Ethernet (DAC/AEC/optical), InfiniBand, RoCEv2 | 5–15 pJ/bit | 0.5–5 µs |
| Pod-level | 10s of m | Ethernet/IB optical (SR/DR) | ~10 pJ/bit | 1–5 µs |
| Datacenter-level | 100s of m | Ethernet optical (FR/LR), optical circuit switching | ~15 pJ/bit | 5–20 µs |
| Campus / metro DCI | 1–80 km | 400G/800G-ZR coherent pluggables | 10s of pJ/bit | 10s of µs |
| Long-haul / regional | 80–3000 km | Coherent DWDM, ROADM, OTN | 100s of pJ/bit | ms |
| Subsea / intercontinental | 1000s of km | Submarine coherent DWDM, repeatered EDFA | 100s of pJ/bit | 10s–100s of ms |

The energy-per-bit column tells a story that drives much of the technology roadmap: every time data crosses a boundary in this hierarchy, the energy cost rises by roughly an order of magnitude, and the latency rises commensurately. The entire thrust of co-packaged optics, chiplet integration, and memory disaggregation is to flatten this hierarchy — to push high-bandwidth, low-energy interconnect outward to longer length scales, so that, for example, a memory pool 10 meters away can be accessed almost as cheaply as memory on the local board.

## Key Industry Bodies and Standards

No single organization governs datacenter networking; instead, a federation of standards bodies, industry consortia, and open-source communities each own a slice of the stack. Understanding who owns what is essential to reading the roadmaps.

- **IEEE 802.3** is the Ethernet working group, responsible for every Ethernet physical-layer and MAC standard from the original 10 Mbps coaxial Ethernet to the emerging 1.6 Terabit Ethernet (Task Force P802.3dj). The IEEE process is consensus-driven, multi-vendor, and slow but durable; an IEEE-ratified Ethernet standard is a near-guarantee of multi-vendor interoperability.

- **IETF (Internet Engineering Task Force)** owns the protocols above the data link layer through its RFC (Request for Comments) process: IP, TCP, UDP, BGP, VXLAN (RFC 7348), GENEVE (RFC 8926), EVPN (RFC 7432), and the influential RFC 7938 ("Use of BGP for Routing in Large-Scale Data Centers") that codified the now-standard practice of running BGP as the datacenter underlay routing protocol.

- **OIF (Optical Internetworking Forum)** defines the electrical and optical interfaces that glue the optical and electrical worlds together: the Common Electrical Interface (CEI) specifications (CEI-112G, CEI-224G) that define chip-to-module and chip-to-chip SerDes, the implementation agreements for coherent pluggables (400ZR), and co-packaging interfaces.

- **CXL Consortium** governs Compute Express Link, having absorbed the earlier Gen-Z, CCIX, and OpenCAPI efforts into a single memory-and-coherence interconnect standard.

- **PCI-SIG (PCI Special Interest Group)** governs PCIe, releasing the generational specifications (PCIe 6.0, 7.0) on a now roughly three-year cadence.

- **UCIe Consortium** governs the Universal Chiplet Interconnect Express die-to-die standard, founded in 2022 by Intel, AMD, Arm, TSMC, Samsung, and others.

- **InfiniBand Trade Association (IBTA)** governs InfiniBand and, importantly, RoCE (RDMA over Converged Ethernet), since RoCE reuses InfiniBand's transport semantics over Ethernet.

- **Open Compute Project (OCP)** is the hyperscaler-led open-hardware community that standardizes server, rack, switch, and optical hardware designs, including the influential work on co-packaged optics, the SONiC NOS ecosystem, and the Open Domain-Specific Architecture (ODSA) chiplet work (Bunch of Wires).

- **JEDEC** governs memory standards including the HBM (High Bandwidth Memory) generations and DDR/LPDDR, which increasingly intersect with networking through CXL-attached memory and HBM as an on-package interconnect.

- **Telecom Infra Project (TIP)** is the Meta-led counterpart to OCP for the optical and telecom transport domain, driving disaggregated optical (Open Optical Packet Transport) and open RAN.

## Market Size and Economics

The global datacenter networking equipment market — switches, routers, and related systems — exceeded **$30 billion** in annual revenue in 2024 and is growing, with an AI-driven premium segment growing far faster than the general-compute segment. Several adjacent markets compound the total addressable opportunity:

- The **optical transceiver market** exceeded **$15 billion** and is among the fastest-growing hardware categories in technology, propelled by the transition from 100G to 400G to 800G and the explosive demand for AI cluster interconnect. A single large AI training cluster can consume hundreds of thousands of high-speed optical transceivers.

- The **switch and routing ASIC (silicon) market** is dominated by Broadcom, with Marvell, NVIDIA, Cisco (Silicon One), and a long tail of custom hyperscaler designs competing for the remainder. Switch silicon is a high-margin, high-moat business: the combination of leading-edge SerDes, packet-processing pipelines, software development kits, and customer lock-in creates a defensible position that has made Broadcom's networking franchise one of the most profitable in semiconductors.

- The **cable and connector market** — direct-attach copper (DAC), active electrical cable (AEC), active optical cable (AOC), and the structured fiber plant — is a multi-billion-dollar business in its own right, increasingly important as the reach limits of passive copper shrink at each speed generation, forcing a transition to active and optical solutions even for short rack-internal links.

The economics of AI have warped these markets. In a traditional cloud datacenter, networking might represent 10–15% of capital expenditure. In an AI training cluster, the networking fraction can be substantially higher when measured properly, because each GPU server demands an order of magnitude more network bandwidth, and because the InfiniBand or high-end Ethernet fabric required to deliver lossless, low-latency, high-bandwidth interconnect commands premium pricing. NVIDIA's networking revenue alone runs at several billion dollars per year, and the company has explicitly positioned the network — not just the GPU — as a strategic control point.

## The Competitive Landscape at a Glance

The vendor landscape, covered in exhaustive detail in File 23, breaks into several overlapping arenas:

- **Switch and router systems**: Cisco (the traditional incumbent, dominant in enterprise and carrier, present in cloud), Arista (the cloud-and-AI-optimized challenger growing rapidly at Cisco's expense), Juniper (strong in service-provider routing, acquired by HPE), and the white-box ecosystem (Dell, Edgecore, and others running SONiC or other disaggregated NOSes on merchant silicon).

- **Switch silicon**: Broadcom (dominant, ~55–60% share, with the Tomahawk, Trident, and Jericho families), Marvell (Teralynx, Prestera, plus a large custom-silicon business and the Inphi coherent DSP franchise), Intel (the now-wound-down Tofino programmable line), and NVIDIA (Spectrum Ethernet and Quantum InfiniBand).

- **AI fabric**: NVIDIA dominant via InfiniBand (a near-monopoly), NVLink/NVSwitch (proprietary scale-up), and Spectrum-X (its Ethernet AI play), challenged by AMD (Ethernet-centric, plus UALink), Intel (Gaudi, Ethernet-centric), and the hyperscalers' custom interconnects (Google TPU ICI, Amazon NeuronLink).

- **Optical systems and components**: Ciena, Nokia, and Infinera in DWDM and DCI systems; Lumentum and Coherent (the merged II-VI/Coherent) in components (WSS, lasers, modulators); Acacia (Cisco), Marvell (Inphi), and others in coherent DSPs; and a large transceiver-module ecosystem spanning Western (Coherent, Lumentum) and Chinese (InnoLight, Eoptolink, Accelink, HiLink) suppliers.

- **Custom silicon at hyperscalers**: Google (TPU interconnect, custom switches), Amazon (Nitro, Trainium NeuronLink), Microsoft (MANA, Azure Boost), and Meta (MTIA accelerator, OCP switches) increasingly design their own networking silicon to reduce dependence on merchant vendors and to optimize for their specific workloads.

## Key Technology Tensions

The remainder of this database returns repeatedly to a handful of structural tensions that will shape the next decade:

**Ethernet versus InfiniBand for AI.** InfiniBand offers native losslessness, lower and more predictable latency, hardware-offloaded collectives (SHARP), and a vertically integrated NVIDIA stack — at the cost of a single-vendor ecosystem, a proprietary management plane, and premium pricing. Ethernet offers an open multi-vendor ecosystem, operational familiarity (BGP, OpenConfig, standard tooling), and ferocious cost competition — at the cost of needing PFC and sophisticated congestion control (DCQCN, and newer schemes from the Ultra Ethernet Consortium) to approximate InfiniBand's lossless behavior. The hyperscalers, with their preference for open ecosystems and operational control, are pushing hard toward Ethernet; NVIDIA's Spectrum-X is its attempt to capture the Ethernet AI fabric with InfiniBand-like performance and NVIDIA-controlled congestion management.

**Electrical versus optical interconnect.** As per-lane signaling rates climb from 50G to 100G to 200G PAM4, the reach of passive copper collapses, and the energy and signal-integrity cost of driving electrical signals from a switch ASIC out to a front-panel pluggable transceiver becomes a dominant fraction of switch power. Linear-drive optics (LPO) and co-packaged optics (CPO) are the industry's responses, pushing the optical engine closer to (and eventually inside) the ASIC package.

**Merchant silicon versus custom ASIC.** Broadcom's merchant switch silicon offers the fastest time-to-market and the broadest ecosystem; custom ASICs (Cisco Silicon One, hyperscaler in-house designs) offer differentiation and workload optimization at enormous engineering cost. The balance shifts back and forth, but the gravitational pull of merchant silicon's economics is strong for all but the very largest operators.

**Proprietary versus open standards.** NVLink versus UALink, InfiniBand versus Ethernet, integrated optical systems versus disaggregated open line systems, vendor CLIs versus OpenConfig — at every layer, a tension exists between the performance and integration of a proprietary, vertically owned solution and the flexibility, multi-sourcing, and cost discipline of an open standard. The history of networking strongly favors open standards in the long run (Ethernet's repeated victories are the canonical example), but the AI era has temporarily rewarded vertical integration, and the resolution of this tension is the central strategic drama of the field.

These tensions are not abstract; they translate directly into multi-billion-dollar product decisions, into the architecture of the clusters that train frontier AI models, and into the competitive fortunes of the companies covered in this database. The chapters that follow examine each layer of the interconnect hierarchy in turn, building from the protocol stack (File 02) through the chip-level interconnects (PCIe, CXL, UCIe), the rack-level fabrics (Ethernet, InfiniBand, RDMA), the optical layer (fundamentals, coherent, switching, DCI, co-packaged optics), the switching silicon, the AI fabric architectures, and finally the vendor landscape, future roadmaps, and the sustainability constraints that increasingly bound the whole enterprise.

## Cross-Cutting Themes: How to Read This Database

Before descending into the layers, it is worth naming the **cross-cutting themes** that recur at every scale, because they are the threads that tie a database spanning micrometers to oceans into a coherent whole. A reader who holds these themes in mind will find the same patterns repeating, in different physical dress, at every layer.

The first theme is the **energy-per-bit hierarchy** (developed quantitatively in File 13). Every interconnect has a characteristic energy cost to move a bit, and these costs rise by roughly an order of magnitude at each step outward in the hierarchy — from sub-picojoule on-die wires to hundreds of picojoules for long-haul coherent. This hierarchy is the deep reason for almost every architectural choice in the database: why GPU-to-GPU communication uses NVLink rather than PCIe, why co-packaged optics exists, why chiplets are integrated, why memory disaggregation is hard, and why the future (File 24) is the *flattening* of this hierarchy.

The second theme is **cross-layer co-design**. The clean layering of the OSI model (File 02) is, in high-performance fabrics, deliberately violated: the physical-layer FEC, the link-layer flow control, the network-layer load balancing, and the transport-layer congestion control are co-designed as a single control system, and the highest performance comes from optimizing across layer boundaries, not within them. This theme recurs as the parallelism-topology-collective co-design of AI training (File 15), the ASIC-optics co-design of co-packaged optics (File 13), and the overlay-underlay co-design of network virtualization (File 20).

The third theme is the **proprietary-versus-open tension**. At every layer, a proprietary, vertically integrated solution (offering peak performance and tight integration) contends with an open, multi-vendor standard (offering flexibility, competition, and lower cost): InfiniBand versus Ethernet, NVLink versus UALink, integrated optical systems versus disaggregated open line systems, vendor CLIs versus OpenConfig, proprietary CPO versus standardized CPO. The history of networking strongly favors open standards in the long run, but the AI era has temporarily rewarded vertical integration, and the resolution of this tension is the central strategic drama traced through the database.

The fourth theme is **the losslessness-and-congestion problem**. Wherever RDMA or high-performance transport appears (Files 06, 07, 08, 17), the network must be lossless or near-lossless, and achieving that without the pathologies of backpressure (PFC deadlock, congestion spreading) is a recurring, hard problem solved differently by InfiniBand (proactive credits) and Ethernet (reactive PFC plus congestion control), and central to whether a fabric performs.

The fifth theme is **disaggregation and composability**. The trajectory at every scale is toward breaking monoliths into pools of independently optimized, independently sourced components reconnected by high-performance fabrics: the chip into chiplets (File 05), the server's memory into CXL pools (File 04), storage into disaggregated NVMe-oF (File 18), and ultimately the datacenter into composable pools of compute, memory, and accelerators (File 24). The network is the enabler of disaggregation — it is only viable because the fabric has become fast and lossless enough to make remote resources behave like local ones.

The sixth and final theme is **the AI economic leverage**. Because the network is a barrier-bound force multiplier on the most expensive resource in the datacenter (the GPU fleet), its efficiency translates directly into the economic return on enormous AI investments (File 15). This leverage — a few percentage points of GPU utilization on a vast fleet being worth more than the entire incremental cost of a superior fabric — is why networking, long treated as plumbing, now commands disproportionate engineering attention and capital, and why this database treats it as a first-order subject. A reader attuned to these six themes — the energy hierarchy, cross-layer co-design, proprietary-versus-open, losslessness, disaggregation, and AI economic leverage — will recognize them recurring, in different physical form, in every chapter that follows.
