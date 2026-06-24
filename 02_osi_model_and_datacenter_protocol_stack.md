# OSI Model, Protocol Stack, and Datacenter Networking Layers

## The OSI Model as a Map of the Datacenter Stack

The seven-layer Open Systems Interconnection (OSI) reference model, formalized by ISO in 1984, is an idealization that no real network implements literally, yet it remains the indispensable shared vocabulary of networking. Its enduring value in the datacenter is as a map: it tells us at which layer a given technology operates, which layers it depends on, and which layers depend on it. When an AI training engineer complains that "the fabric is dropping packets," when a data-center architect debates "Layer 2 versus Layer 3 to the host," or when an optical engineer specifies a "Layer 1 retimer," they are all locating their concern on this map.

This chapter walks the model from the bottom up, populating each layer with the specific protocols, encodings, and hardware that constitute a modern datacenter network. The crucial insight — developed at the end of the chapter — is that in a high-performance AI fabric, the layers do not operate independently. The physical-layer forward error correction, the link-layer flow control, the network-layer load balancing, and the transport-layer congestion control are co-designed and tightly coupled. A change at one layer ripples through all the others.

The seven layers, briefly:

1. **Physical** — the transmission of raw bits over a medium (copper, fiber, backplane). Concerns: signaling, modulation, encoding, forward error correction, connectors, reach.
2. **Data Link** — framing, addressing within a single link or LAN, and media access. Concerns: Ethernet frames, MAC addresses, VLANs, flow control.
3. **Network** — addressing and routing across multiple links to reach any destination. Concerns: IP, BGP, ECMP, overlays.
4. **Transport** — end-to-end delivery, reliability, and congestion control. Concerns: TCP, UDP, RDMA transport, congestion-control algorithms.
5. **Session** — establishing, managing, and tearing down conversations.
6. **Presentation** — data representation, encryption, serialization.
7. **Application** — the protocols applications speak: HTTP, gRPC, NFS, NVMe-oF, gNMI.

In practice, the TCP/IP model collapses layers 5–7 into a single "application" layer, and most datacenter discussion follows suit. We retain the OSI numbering because it precisely names the boundaries that matter for hardware offload and protocol design.

## Layer 1: The Physical Layer in the Datacenter

The physical layer is where the abstract bit meets the laws of physics, and in the datacenter it is dominated by three media classes — copper, active electrical, and optical — and by an ongoing battle against the signal degradation that intensifies at every speed increase.

### Copper, Active Electrical, and Optical Media

**Direct-Attach Copper (DAC)**, also called twinax, is a passive shielded copper cable used for the shortest links — typically top-of-rack switch to server within the same rack, up to a few meters. DAC is the cheapest and lowest-power option (no active components, no lasers), but its reach shrinks dramatically with each speed generation. At 10G and 25G per lane, passive DAC reached 3–5 meters; at 50G PAM4 it reached ~2–3 meters; at 100G PAM4 passive DAC is limited to roughly 1.5–2 meters; and at 200G per lane passive copper becomes marginal even within a rack.

**Active Electrical Cable (AEC)** embeds signal-conditioning chips (retimers or redrivers) in the cable connectors to extend copper reach. By regenerating the electrical signal, AEC pushes the usable distance back out to ~7 meters even at high per-lane rates, at the cost of added power (a few watts) and cost. AECs have become essential for rack-to-rack and within-row links as passive DAC runs out of reach; companies such as Credo built substantial businesses on AEC for hyperscale AI deployments.

**Optical media** take over wherever copper cannot reach. The taxonomy of optical reaches is encoded in IEEE nomenclature:
- **SR (Short Reach)** — multimode fiber with VCSEL lasers, typically 850 nm, reaching 30–100 meters; used intra-rack and intra-row. Variants: SR4 (4 lanes), SR8 (8 lanes).
- **DR (500 m reach)** — single-mode fiber, single-wavelength per lane, ~500 meters; used inter-rack within a pod. DR4 = four parallel single-mode fibers.
- **FR (2 km)** — single-mode, ~2 km; cross-pod within a campus.
- **LR (10 km)** — single-mode, 10 km; campus and short DCI.
- **ER (40 km)** and **ZR/ZR+ (80 km and beyond)** — extended reach and coherent, for datacenter interconnect.

### Signaling: NRZ, PAM4, and the March to PAM8

For decades, datacenter SerDes used **NRZ (Non-Return-to-Zero)** signaling: two voltage levels encoding one bit per symbol (one baud). NRZ is robust — its two levels are far apart, giving good noise immunity — but its spectral efficiency is only one bit per baud, so doubling the data rate requires doubling the baud rate, which runs into the bandwidth limits of the channel.

At 50 Gbps per lane and above, the industry transitioned to **PAM4 (4-level Pulse Amplitude Modulation)**, which uses four voltage levels to encode two bits per symbol. PAM4 halves the required baud rate for a given data rate (a 100 Gbps PAM4 lane runs at ~53.1 Gbaud rather than ~106 Gbaud for NRZ), keeping the signal within the channel's bandwidth. The cost is severe: with four levels packed into the same voltage swing, the spacing between levels is one-third that of NRZ, costing approximately **9.5 dB of signal-to-noise ratio**. This SNR penalty is why PAM4 mandates forward error correction — the raw bit error rate before correction is far too high to use directly.

**PAM8 (8 levels, 3 bits per symbol)** has been researched for beyond-200G-per-lane signaling but faces an even more brutal SNR penalty and demands data converters with extreme effective-number-of-bits (ENOB) at enormous bandwidths. The industry consensus as of the mid-2020s is that PAM8 is impractical for production; the path to higher per-lane rates runs through higher baud rates and, ultimately, optics, rather than through more modulation levels.

### Forward Error Correction

**Forward Error Correction (FEC)** adds redundant symbols to the data stream so the receiver can detect and correct errors without retransmission. In high-speed datacenter links, FEC is not optional cleanup — it is a fundamental enabler that allows PAM4 to operate at a raw (pre-FEC) bit error rate as high as 10⁻⁴ while delivering a post-FEC BER of 10⁻¹² or better to the upper layers.

The dominant codes are Reed-Solomon:
- **RS(528,514)**, the "Base-R" or "KR4/CR4" FEC, adds 14 parity symbols to 514 data symbols (2.7% overhead) and corrects up to 7 symbol errors per codeword. It was standardized for 25G NRZ links and adds roughly 80 nanoseconds of latency.
- **RS(544,514)**, the "KP4" FEC, adds 30 parity symbols (5.8% overhead), corrects up to 15 symbol errors, and is the workhorse FEC for 50G PAM4 and faster lanes. Its latency is roughly 100–150 nanoseconds.

FEC latency matters intensely for AI and storage RDMA, because it adds directly to every link traversal and therefore to the round-trip time that bounds collective-operation completion. This tension has produced specialized **low-latency FEC (LL-FEC)** schemes from the Ethernet Technology Consortium that trade some correction strength for reduced latency, and it occasionally leads operators to disable FEC entirely on very short, very clean links — a risky optimization, since an uncorrected error on an RDMA flow can be far more costly than the FEC latency it saves.

The figures of merit at this layer are the **eye diagram** (a visualization of signal quality showing the "eye opening" between symbol levels), the **bit error rate (BER)** target (10⁻¹² post-FEC for Ethernet; pre-FEC budgets around 10⁻⁴ for KP4-protected PAM4), and the channel **insertion loss budget** in decibels, which determines whether a given reach is achievable with a given combination of signaling, equalization, and FEC.

## Layer 2: The Data Link Layer

The data link layer frames bits into addressable units and governs access to the shared medium. In the datacenter it is overwhelmingly Ethernet (with InfiniBand's link layer as the major alternative, covered in File 07).

### The Ethernet Frame

An Ethernet frame, as it appears on the wire, comprises:
- **Preamble (7 bytes)** and **Start Frame Delimiter (SFD, 1 byte)** — a known pattern that lets the receiver synchronize its clock and find the frame boundary.
- **Destination MAC address (6 bytes)** and **Source MAC address (6 bytes)** — 48-bit hardware addresses; the most significant bit of the destination distinguishes unicast from multicast, and a separate bit distinguishes globally unique (OUI-assigned) from locally administered addresses.
- **EtherType / Length (2 bytes)** — values ≥ 0x0600 indicate the payload protocol (0x0800 = IPv4, 0x86DD = IPv6, 0x8100 = 802.1Q VLAN tag, 0x8847 = MPLS); smaller values indicate frame length (legacy 802.3).
- **Payload (46–1500 bytes standard, up to ~9000 bytes for jumbo frames)** — the encapsulated upper-layer packet. Jumbo frames are universal in datacenters because they amortize per-frame overhead and reduce CPU interrupt load for bulk transfers.
- **Frame Check Sequence (FCS, 4 bytes)** — a CRC-32 over the frame for error detection (note: detection, not correction; that is the physical-layer FEC's job).

### VLANs, QinQ, and Flow Control

**802.1Q VLAN tagging** inserts a 4-byte tag after the source MAC, containing a 12-bit VLAN ID (4,096 VLANs) and a 3-bit Priority Code Point (PCP, the "class of service" used by Priority Flow Control). **802.1ad (QinQ)** stacks two tags — an outer "service" tag and an inner "customer" tag — to scale segmentation beyond 4,096 and to let providers transport customer VLANs transparently. In modern datacenters, VLAN-based segmentation at scale has largely been superseded by VXLAN overlays (below), but VLAN tagging remains ubiquitous for local segmentation and, critically, for carrying the priority bits that drive lossless behavior.

Flow control is where the data link layer becomes essential to AI fabrics:
- **802.3x PAUSE** is the original link-level flow control: when a receiver's buffer fills, it sends a PAUSE frame telling the sender to stop transmitting for a specified time. PAUSE is blunt — it halts *all* traffic on the link, which is unacceptable when latency-sensitive control traffic shares a link with bulk data.
- **802.1Qbb Priority-based Flow Control (PFC)** refines PAUSE to operate per-priority: it can pause priority class 3 (say, RDMA traffic) while letting other classes flow. PFC is the mechanism that makes "lossless Ethernet" possible, because it lets a switch backpressure an upstream sender before its buffer overflows and drops packets. PFC is also the source of much pathology — head-of-line blocking, congestion spreading, and the dreaded PFC deadlock and PFC storms — examined in depth in Files 06 and 17.
- **802.1Qaz Enhanced Transmission Selection (ETS)** and the **Data Center Bridging Exchange (DCBX, part of 802.1AB/LLDP)** allow switches and NICs to negotiate bandwidth allocation among priority classes and to exchange PFC and ETS configuration automatically, ensuring both ends of a link agree on which priorities are lossless.

**802.3ad Link Aggregation (LACP)** bonds multiple physical links into one logical link for bandwidth and redundancy, distributing flows across members by hashing. **LLDP (Link Layer Discovery Protocol, 802.1AB)** lets devices advertise their identity and capabilities to neighbors, underpinning automated topology discovery and DCBX.

## Layer 3: The Network Layer

The network layer provides global addressing and routing. In the datacenter, it is IP (v4 and increasingly v6) for addressing and BGP for routing the underlay.

### IPv4 and IPv6 Headers

The **IPv4 header** is 20 bytes (without options): version and header length, Type of Service / DSCP (Differentiated Services Code Point) and ECN bits, total length, identification/flags/fragment offset (for fragmentation), TTL (time to live, decremented at each hop to kill loops), protocol (6 = TCP, 17 = UDP), header checksum, and 32-bit source and destination addresses. The two **ECN (Explicit Congestion Notification)** bits in the ToS field are central to datacenter congestion control: a congested switch can set the "Congestion Experienced" codepoint instead of dropping the packet, signaling the endpoints to slow down without loss.

The **IPv6 header** is a fixed 40 bytes with a cleaner design: version, traffic class (DSCP+ECN), flow label (a 20-bit field that can carry per-flow entropy for load balancing), payload length, next header (replacing both "protocol" and the options mechanism via a chain of extension headers), hop limit (the TTL equivalent), and 128-bit source and destination addresses. The flow label is increasingly used in large datacenters to improve ECMP hashing entropy, and the vast address space simplifies addressing at hyperscale.

### BGP, OSPF, and IS-IS in the Datacenter

The defining architectural decision of the modern datacenter underlay is the use of **BGP (Border Gateway Protocol)** — historically the protocol of the global internet — as the *intra*-datacenter routing protocol. **RFC 7938, "Use of BGP for Routing in Large-Scale Data Centers,"** codified this practice. The logic: a Clos fabric is a regular, densely meshed topology where every leaf connects to every spine. Running a link-state protocol (OSPF or IS-IS) across such a topology produces enormous link-state databases and frequent flooding; BGP, by contrast, advertises only reachability, scales to hundreds of thousands of prefixes, supports rich policy, and — crucially — provides **ECMP (Equal-Cost Multi-Path)** naturally, since a leaf learns the same destination prefix via every spine and load-balances across all of them.

In the canonical design, each switch is its own BGP Autonomous System (using private or 4-byte ASNs), eBGP sessions run on every fabric link, and the entire fabric becomes a routed Layer 3 network from the top of rack down — sometimes all the way to the host ("routing on the host"). **OSPF and IS-IS** still appear as underlay protocols, particularly in service-provider-influenced designs and in some optical/MPLS underlays, and **route reflectors** are used where iBGP is preferred to reduce the full-mesh session count. But the BGP-everywhere model, often automated and templated, is the hyperscale default.

ECMP's interaction with overlays is a recurring theme: because VXLAN encapsulates the inner packet inside an outer UDP/IP header, the fabric load-balances on the *outer* header, and the encapsulating device deliberately varies the outer UDP source port to inject entropy so that different inner flows hash to different paths. Poorly randomized entropy is a classic cause of ECMP imbalance and the "elephant flow" hot-spotting that plagues RDMA fabrics (File 17).

## Layer 4: The Transport Layer

The transport layer provides end-to-end delivery and, in the datacenter, is the home of the congestion-control algorithms that determine whether a fabric performs well or collapses under load.

### TCP and Its Datacenter Congestion-Control Variants

TCP provides reliable, ordered, byte-stream delivery with congestion control. The classic algorithm, **CUBIC**, increases its sending window aggressively and backs off on loss; it works well on the wide-area internet but reacts poorly to the shallow buffers and microsecond RTTs of the datacenter. Several datacenter-specific variants emerged:
- **DCTCP (Data Center TCP)** uses ECN marking proportionally: rather than halving the window on any congestion signal, it estimates the *fraction* of marked packets and reduces the window in proportion, keeping switch queues short and latency low. DCTCP is the conceptual ancestor of DCQCN.
- **BBR (Bottleneck Bandwidth and Round-trip propagation time)**, from Google, models the bottleneck bandwidth and minimum RTT directly and paces transmission to operate at the optimal point, rather than reacting to loss. BBR is widely used for Google's WAN and serving traffic.

### UDP, RDMA, and QUIC

For RDMA over Ethernet, the transport is **UDP**: **RoCEv2** encapsulates InfiniBand transport semantics inside a UDP/IP packet (destination port 4791), making RDMA routable across a Layer 3 fabric. Because UDP itself provides no congestion control, RoCEv2 relies on a combination of PFC (link-layer losslessness) and a congestion-control algorithm — **DCQCN, HPCC, Timely, or Swift** — implemented in the NIC and signaled via ECN or in-band telemetry. These are examined in detail in Files 06, 08, and 17. The key concept here is **802.1Qau Quantized Congestion Notification (QCN)**, an early Layer 2 congestion-notification scheme whose ideas live on in DCQCN's quantized rate adjustments.

**QUIC**, originally a Google protocol and now an IETF standard (RFC 9000), runs reliable, multiplexed, encrypted streams over UDP. In the datacenter it matters chiefly for front-end and inter-service serving traffic rather than for the AI fabric, but its user-space implementation and head-of-line-blocking avoidance make it increasingly relevant for east-west microservice communication.

## Layers 5–7: Session, Presentation, and Application

Above the transport layer sit the protocols that applications and infrastructure actually speak. In the datacenter, the most important are:

- **Storage protocols**: **NFS** (network file system) for shared files; **iSCSI** (SCSI block commands over TCP) for block storage; and, most importantly for modern flash, **NVMe-oF (NVMe over Fabrics)**, which carries NVMe's queue-pair storage model over RDMA, TCP, or Fibre Channel transports (File 18). NVMe-oF over RDMA achieves sub-20-microsecond remote storage latency, approaching local NVMe.
- **RPC and APIs**: **gRPC** (HTTP/2-based, protobuf-serialized remote procedure calls) and **REST/HTTP** dominate service-to-service and management communication.
- **Network management and telemetry**: **OpenConfig** (vendor-neutral YANG data models), **gNMI (gRPC Network Management Interface)** for streaming telemetry and configuration, and **gRPC-based telemetry** pipelines that push per-port counters, buffer occupancy, and congestion events to collectors at millisecond granularity, replacing the legacy SNMP polling model (File 16, File 22).

## How the Layers Interact in a Lossless AI Fabric

The central thesis of this chapter is that, in a high-performance AI training fabric, the OSI layers are not independent — they are a tightly coupled control system, and understanding the coupling is essential to understanding why these fabrics are hard to build.

Consider what happens during a single AllReduce on a RoCEv2 Ethernet fabric:

1. **Physical layer**: every link runs PAM4 with RS(544,514) FEC. The FEC adds ~150 ns of latency per hop and guarantees a post-FEC BER low enough that the RDMA transport effectively never sees a corruption-induced loss. If FEC were weaker, corruption losses would trigger RDMA retransmissions; if FEC latency were higher, the collective would complete more slowly.

2. **Data link layer**: RDMA traffic rides on a dedicated priority class (say PCP 3) configured for PFC. When a switch's buffer for that priority approaches full, it sends a PFC PAUSE upstream, preventing the buffer from overflowing and dropping an RDMA packet. This is what makes the fabric "lossless." But PFC is a hop-by-hop backpressure: if it triggers deep in the fabric, it propagates upstream, and if the topology has cyclic buffer dependencies, it can deadlock — so the network layer's topology and the link layer's PFC configuration must be co-designed to be deadlock-free.

3. **Network layer**: the AllReduce traffic is spread across the Clos fabric by ECMP. Because RDMA bulk transfers between a given pair of endpoints often share a single queue pair (and thus a single 5-tuple), naive ECMP hashes them all onto one path, creating "elephant flow" hot spots. Mitigations live at this layer (flowlet switching, adaptive routing, per-packet spraying with reordering-tolerant transport) and interact directly with the transport layer's tolerance for out-of-order delivery.

4. **Transport layer**: the RoCEv2 NIC runs DCQCN (or a newer algorithm). When a switch experiences congestion, it sets the ECN "Congestion Experienced" bit; the receiver reflects this to the sender via a Congestion Notification Packet; the sender reduces its injection rate. DCQCN's job is to keep queues short enough that PFC rarely triggers — because while PFC prevents loss, it does so by spreading congestion, so a well-tuned fabric uses ECN/DCQCN as the primary, graceful control and PFC only as the last-resort safety net.

The art of building these fabrics, explored throughout this database, lies in tuning all four layers together: FEC strength versus latency, PFC thresholds versus ECN marking thresholds, buffer sizing versus burst absorption, ECMP entropy versus flow collisions, and DCQCN parameters versus message-size distribution. Get the coupling wrong and the fabric either drops RDMA packets (catastrophic for throughput) or spreads PFC backpressure until the whole fabric stalls (catastrophic for the collective). Get it right and the fabric delivers near-wire-rate, near-deterministic collective performance across tens of thousands of GPUs. This is why AI networking is a systems problem, not a layer-by-layer problem.

## Overlay Networks: VXLAN, GENEVE, and GRE

The final piece of the protocol stack is the overlay — the virtual networks built on top of the physical (underlay) fabric to provide tenant isolation, mobility, and addressing independence.

**VXLAN (Virtual Extensible LAN, RFC 7348)** is the dominant datacenter overlay. It encapsulates a complete Layer 2 Ethernet frame inside an outer UDP/IP packet. The VXLAN header is 8 bytes and carries a **24-bit VNI (VXLAN Network Identifier)**, providing 16 million virtual network segments — three orders of magnitude beyond the 4,096 of VLANs. The encapsulation is performed by a **VTEP (VXLAN Tunnel Endpoint)**, which can live in a hypervisor virtual switch, in a NIC (hardware VTEP, offloaded), or in a physical switch (hardware VTEP in the ASIC). The total overhead is roughly 50 bytes (14-byte outer Ethernet + 20-byte outer IPv4 + 8-byte UDP + 8-byte VXLAN), which is why jumbo frames and hardware offload are essential — software VXLAN encapsulation at line rate would consume enormous CPU. VXLAN's control plane in modern deployments is **BGP EVPN (Ethernet VPN, RFC 7432)**, which distributes MAC and IP reachability among VTEPs, supports multi-homing, and enables symmetric integrated routing and bridging (IRB).

**GENEVE (Generic Network Virtualization Encapsulation, RFC 8926)** generalizes VXLAN with an extensible, variable-length header carrying type-length-value (TLV) options, designed so that SDN controllers can attach arbitrary metadata to tunneled packets. It is favored by software-defined platforms such as VMware NSX-T and is supported by Open vSwitch.

**GRE (Generic Routing Encapsulation)** is an older, simpler tunneling protocol still used for specific point-to-point tunnels and in some cloud gateway designs.

The key engineering consideration across all overlays is **hardware offload**: encapsulation and decapsulation, and the entropy generation needed for the underlay's ECMP to load-balance tunneled traffic, must happen in the NIC or switch ASIC at line rate. A VTEP that varies the outer UDP source port as a hash of the inner flow lets the underlay spread tunneled flows across all ECMP paths — the same entropy mechanism that, done poorly, causes the load imbalance discussed at Layer 3. The overlay and underlay, like every other pair of layers in the AI-era datacenter, must be designed together.

## The Stack in Practice: Encapsulation Depth and Header Overhead

It is worth tracing concretely how deep the encapsulation stack can become in a modern datacenter, because the layering described above composes, and each layer adds header overhead that the hardware must process at line rate. Consider an RDMA write from one GPU server to another, in a multi-tenant cloud, over a VXLAN overlay: the innermost payload is the application data; wrapped in an InfiniBand transport header (BTH) for RDMA; wrapped in a UDP header (port 4791) and an IP header for RoCEv2 routability; wrapped in an inner Ethernet frame; encapsulated by the VTEP in a VXLAN header, an outer UDP header (with entropy in the source port), an outer IP header, and an outer Ethernet frame; and finally framed at the physical layer with preamble, FEC, and the line code. A single packet thus carries half a dozen nested headers, each added by a different layer, and the NIC and switches must parse, process, and (for some layers) modify all of them at hundreds of gigabits per second. This is why hardware offload (parsing, encapsulation/decapsulation, checksum, FEC) and jumbo frames (to amortize the fixed per-packet header overhead over a larger payload) are not optional niceties but fundamental requirements — a software stack processing this encapsulation depth at line rate would consume enormous CPU, and the per-packet header overhead on small frames would waste significant bandwidth. The composability of the protocol stack is powerful, but it imposes a processing and overhead burden that only hardware acceleration and careful frame sizing can bear.

## The Stack in Practice: Why Layer Violations Are Common and Necessary

A final observation about the protocol stack in the datacenter is that the clean layer separation of the OSI model is routinely, and necessarily, **violated** in high-performance designs — and understanding why illuminates the engineering reality. The model says each layer should be independent, communicating only through well-defined interfaces, but datacenter performance demands cross-layer optimization. RDMA (a transport/session concern) reaches down to require physical-layer FEC and link-layer PFC (losslessness). Congestion control (transport) depends on switches setting ECN (network/link) and, in HPCC, on switches embedding telemetry (a deliberate layer violation that exposes physical queue state to the transport). ECMP (network) load-balancing depends on entropy that the overlay (a higher layer) must deliberately inject into the outer header. In-network computing (SHARP) has switches (a network-layer device) performing application-layer reduction operations. Each of these "violations" is a deliberate cross-layer optimization that trades the model's clean modularity for the performance the workload demands.

This is not sloppiness; it is the central engineering principle that this database returns to repeatedly: in the AI-era datacenter, the layers form a tightly coupled control system, and the highest performance comes from co-designing across layer boundaries — FEC with transport, PFC with congestion control, overlay with underlay, network with collective communication. The OSI model remains the indispensable *map* (it names the concerns and their natural boundaries), but the *territory* of a high-performance fabric is one of deliberate, careful cross-layer coupling. Holding both ideas at once — the model as a map, and cross-layer co-design as the reality — is essential to understanding how datacenter networks actually achieve their performance, and it is the lens through which the remainder of this database examines each technology.
