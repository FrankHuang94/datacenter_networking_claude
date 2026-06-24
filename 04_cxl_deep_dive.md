# CXL (Compute Express Link) — Architecture, Protocol, Memory Semantics, and Roadmap

## Introduction: The Interconnect That Redefines Memory

Compute Express Link (CXL) is the most consequential new interconnect standard of the 2020s, because it attacks a problem that no amount of compute scaling can solve on its own: the **memory wall**. As processors gained cores and accelerators gained FLOPS at a furious pace, the memory subsystem — bounded by the number of DIMM slots a CPU can drive, the bandwidth of DDR channels, and the cost of DRAM — fell progressively further behind. CXL is the industry's coordinated answer. It turns the humble PCIe link into a cache-coherent, memory-semantic fabric, allowing memory to be expanded beyond the DIMM slots, shared across hosts, pooled and reallocated dynamically, and accessed coherently by accelerators. In doing so, CXL begins to dissolve the boundary between "an interconnect" and "the memory system," and it lays the foundation for the disaggregated, composable datacenter that File 24 projects into the next decade.

This chapter develops CXL from its origins and strategic rationale, through its three sub-protocols and three device types, its switching and memory-pooling capabilities, the fabric and peer-to-peer extensions of CXL 3.0/3.1, the dense competitive landscape of CPU vendors and memory makers racing to build CXL products, and the software stack that makes CXL memory usable. CXL is built on PCIe (File 03), so that chapter is prerequisite; CXL also intersects with the AI fabric (File 15) and the future roadmap (File 24).

## CXL Origins and Strategic Importance

### The Founding and the Great Consolidation

CXL was introduced by **Intel in 2019**, which contributed an initial specification and formed the CXL Consortium with founding members including **Alibaba, Cisco, Dell EMC, Facebook (Meta), Google, HPE, Huawei, and Microsoft** — a roster spanning the CPU vendor, the hyperscalers, and the system OEMs, signaling broad industry buy-in from the outset. CXL 1.0 was released that year, with CXL 1.1 following shortly to refine the specification.

What makes CXL's history remarkable is the **great consolidation** that followed. Before CXL, the industry had at least three competing coherent/memory-semantic interconnect efforts:
- **CCIX (Cache Coherent Interconnect for Accelerators)**, an Arm-led effort to add cache coherence across PCIe-class links;
- **Gen-Z**, a memory-semantic fabric backed by HPE, Dell, and others, aimed squarely at memory disaggregation and pooling;
- **OpenCAPI (Open Coherent Accelerator Processor Interface)**, an IBM-led coherent interconnect derived from its POWER architecture's CAPI.

Each had merit, but a fragmented ecosystem benefits no one — chip designers cannot afford to support multiple incompatible coherent fabrics, and memory and device vendors need a single large market. Between 2020 and 2022, **Gen-Z, CCIX, and OpenCAPI all transferred their specifications and assets to the CXL Consortium**, and CXL emerged as the single, unified industry standard for cache-coherent and memory-semantic interconnect. This consolidation is one of the cleaner examples of the industry choosing convergence over fragmentation, and it gave CXL an unusually strong starting position.

### Why CXL: The Memory Capacity and Bandwidth Walls

The motivation for CXL is best understood as a set of concrete, painful constraints:

- **The memory capacity wall.** A modern server CPU can directly attach only a limited number of DIMMs — typically 8 to 12 DDR5 channels, each with one or two DIMMs. Once those slots are full, the only way to add more memory to that CPU is... there isn't one, short of buying a bigger CPU socket. Yet many workloads — in-memory databases, large-scale analytics, and especially AI inference and training with large model and KV-cache footprints — are memory-capacity-bound. CXL memory expanders attach additional DRAM (or persistent memory) to the CPU over a CXL link, breaking the DIMM-slot ceiling.

- **The memory bandwidth wall.** DDR5 with 8–12 channels delivers on the order of 300–500 GB/s of CPU memory bandwidth — a figure dwarfed by the multi-terabyte-per-second HBM bandwidth on a GPU. CXL adds memory bandwidth in parallel with the DDR channels: a CXL-attached memory device contributes its own bandwidth, and multiple CXL devices aggregate. While a single CXL link's bandwidth is bounded by the PCIe lanes it runs over, the architecture allows scaling bandwidth by adding links and devices.

- **Coherence for heterogeneous systems.** Modern servers are no longer CPU-only; they pack GPUs, FPGAs, smart NICs, and domain-specific accelerators, all of which need to share data with the CPU. Without coherence, sharing data between a CPU and an accelerator requires explicit, software-managed copies and cache flushes — slow and error-prone. CXL provides hardware cache coherence between the CPU and attached devices, so an accelerator can read and write host memory (and vice versa) with the hardware maintaining a consistent view, dramatically simplifying heterogeneous programming.

### Built on PCIe, Backward Compatible

A crucial design decision underlies CXL's rapid adoption: it is **built on the PCIe physical layer**. CXL uses the same electrical signaling, the same connectors, and the same lanes as PCIe. CXL 1.1 and 2.0 operate over PCIe 5.0 (32 GT/s) lanes; CXL 3.0 operates over PCIe 6.0 (64 GT/s) lanes. At link initialization, a port negotiates whether to operate as a plain PCIe link or as a CXL link via an "alternate protocol negotiation." This means CXL reuses the entire PCIe ecosystem — connectors, retimers, board design, signal-integrity engineering — and a CXL device can fall back to appearing as a PCIe device to a legacy operating system that does not understand CXL. This backward compatibility removed an enormous barrier to adoption: system designers did not have to invent a new physical infrastructure; they reused PCIe.

### Versioning Overview

CXL has evolved through several generations, each adding major capability:
- **CXL 1.0 / 1.1 (2019)** — the foundational release, defining the three sub-protocols and device Types 1 and 2 (accelerators), and Type 3 (memory) in basic form, over a direct point-to-point link.
- **CXL 2.0 (2020)** — added **single-level switching** and **memory pooling**, allowing multiple hosts to share a pool of Type 3 memory devices through a CXL switch, plus support for persistent memory and integrity/encryption.
- **CXL 3.0 (2022)** — a major leap to **PCIe 6.0** physical layer (doubling bandwidth), **multi-level switching and fabrics**, **peer-to-peer** device communication, enhanced coherence with back-invalidation, and far larger scale.
- **CXL 3.1 (2023)** — refinements to the fabric, security (trusted execution environment support), and manageability.

## The Three CXL Sub-Protocols

CXL multiplexes three distinct sub-protocols over a single link, dynamically interleaving them: **CXL.io**, **CXL.cache**, and **CXL.mem**. A given device uses some subset of these depending on its type. The protocols are multiplexed on the wire by the CXL flex-bus logical layer, which interleaves CXL.io traffic (less latency-sensitive) with CXL.cache and CXL.mem traffic (highly latency-sensitive) using a fixed-format, low-latency framing.

### CXL.io — The I/O and Control Path

**CXL.io** is functionally equivalent to PCIe: it maps directly onto PCIe TLP/DLLP semantics and is used for device discovery, configuration, interrupts, register access, and DMA — the control and management path. Every CXL device must support CXL.io, because it provides the mechanism for enumerating and configuring the device just as PCIe does. CXL.io is what guarantees backward compatibility: a CXL device's CXL.io path is indistinguishable from a PCIe device to enumeration software. It is the least latency-sensitive of the three sub-protocols, carrying the housekeeping rather than the hot data path.

### CXL.cache — Device Caching of Host Memory

**CXL.cache** is the sub-protocol that lets an **accelerator (device) coherently cache host CPU memory**. This is the capability that distinguishes CXL from plain PCIe: an accelerator can pull host memory lines into its own cache, operate on them, and have the hardware keep everything coherent with the CPU's caches, without software-managed flushes.

CXL.cache defines two channel directions, each carrying three message classes:
- **D2H (Device-to-Host)**: Request (the device requests a cache line, e.g., for read or for ownership), Response, and Snoop response (the device's reply to a host snoop).
- **H2D (Host-to-Device)**: Request, Response, and Snoop (the host snoops the device's cache to maintain coherence).

Coherence operates at **cache-line granularity (64 bytes)**, matching the CPU's cache-line size. The protocol carries the familiar coherence transactions: reads that request a shared copy (RdShared) or an exclusive/owned copy (RdOwn), writes and evictions (clean eviction, dirty eviction), and snoops that the host issues to invalidate or downgrade lines the device holds (SnpData, SnpInv, SnpCur). The device's coherence state machine maintains MESI-style states for the lines it caches.

A particularly important concept is **bias modes**, which optimize coherence overhead based on access patterns:
- **Host Bias**: the host is treated as the primary owner/manager of a cache line. Device accesses to such lines may require coordination with the host. This is appropriate when the CPU is actively sharing the data.
- **Device Bias**: the accelerator is treated as the owner, and it can access the line with reduced coherence overhead (no need to involve the host on every access). This dramatically reduces latency for GPU- or accelerator-heavy access patterns, where the device is operating on data the CPU is not currently touching. Software (or the runtime) can flip a region between host bias and device bias as the access pattern shifts.

CXL.cache thus enables an accelerator to perform **coherent peer-to-peer DMA** — directly accessing host memory with hardware coherence rather than software-managed copies — which is transformative for the programming model of heterogeneous systems.

### CXL.mem — Host Access to Device-Attached Memory

**CXL.mem** is the inverse direction: it lets the **host CPU access memory that is physically attached to a CXL device**, as if that memory were part of the CPU's own address space. This is the protocol behind CXL memory expanders. The device's memory is exposed as **HDM (Host-managed Device Memory)** and mapped directly into the CPU's physical address space, so ordinary CPU loads and stores reach it (with additional latency).

CXL.mem carries memory transactions — reads, writes, and ownership/sharing operations (the spec defines messages such as MemRd, MemWr, and ownership/share variants) — and maintains coherence. There are two principal HDM models:
- **HDM-H (Host-only coherent)**: the host manages coherence entirely; the device's memory is simply a coherent memory target that the host's home agent controls. This is the model for a simple Type 3 memory expander where the device does not itself cache the memory.
- **HDM-DB (Device-coherent with Back-invalidation)**: the device can cache lines of its own memory (or participate in coherence more actively) and uses **back-invalidation** to invalidate copies that the host holds. This enables device-managed coherence, where a memory device or near-memory accelerator can take a more active role, important for computational memory and for the fabric coherence of CXL 3.0.

The CPU maintains standard MESI coherence states for CXL-attached memory, integrating it into the same coherence domain as local DRAM — which is exactly what makes CXL memory usable by unmodified software, albeit with NUMA-aware optimization (below).

## CXL Device Types

CXL defines three device types based on which sub-protocols they implement, each targeting a distinct use case:

**Type 1 — Accelerator with cache, no device memory (CXL.io + CXL.cache).** A Type 1 device is an accelerator that needs to coherently cache host memory but has no significant local memory of its own to expose to the host. Examples include a smart NIC with a coherent cache or an FPGA accelerator that operates on host data structures. The device caches host memory coherently (via CXL.cache) and is configured/controlled via CXL.io. Type 1 is the right model when the accelerator's job is to chew on data that lives in host memory.

**Type 2 — Accelerator with both cache and device memory (CXL.io + CXL.cache + CXL.mem).** A Type 2 device is a full accelerator — GPU-class — that has its own substantial local memory **and** wants coherent access to host memory. It uses all three sub-protocols: CXL.io for control, CXL.cache to coherently access host memory, and CXL.mem to expose its own local memory to the host coherently. This is the model for accelerators that need tight, bidirectional coherent sharing with the CPU — for example, a GPU whose HBM and the host's DRAM form a unified coherent space. NVIDIA's H100 NVL and various accelerators support CXL in this role, and Intel's Habana Gaudi family and others target Type 2 semantics. Type 2 is the most demanding device class, requiring the full coherence machinery.

**Type 3 — Memory expander, no compute, no caching (CXL.io + CXL.mem).** A Type 3 device is pure memory: DRAM or persistent memory that adds capacity (and bandwidth) to the host, with no compute and no caching of host memory. The host accesses it via CXL.mem; CXL.io handles enumeration and management. Type 3 is the device class driving the memory-expansion and memory-pooling use cases, and it is where the memory vendors are concentrating their products: **Samsung's CMM-D (CXL Memory Module-DRAM), SK Hynix's CMM-H, and Micron's CXL memory expanders** are all Type 3 devices, as are persistent-memory variants and computational-memory switches. Type 3 is the simplest device class to build (no coherence engine to cache host memory) and the most immediately deployable, which is why it leads CXL's commercial adoption.

## CXL Memory Pooling and Switching (CXL 2.0+)

CXL 1.1 supported only a direct point-to-point link between a host and a device. **CXL 2.0 introduced switching**, and with it the transformative capability of **memory pooling**.

### The CXL Switch and Logical Devices

A **CXL switch** sits between multiple hosts and multiple Type 3 memory devices, allowing the memory to be **shared as a pool** rather than dedicated to a single host. CXL 2.0 supports a single switch level; CXL 3.0 extends to multi-level fabrics. The key abstractions are:
- **Logical Device (LD)**: a portion of a physical memory device's capacity, assigned to a particular host. A host sees its LD as memory in its address space.
- **MLD (Multiple Logical Device)**: a single physical Type 3 device partitioned into multiple LDs, each assigned to a different host. One physical memory module thus serves several hosts, each owning a slice. This is the heart of memory pooling — a pool of physical memory carved into LDs and parceled out to hosts on demand.
- **Fabric Manager (FM)**: the software/firmware control plane that manages the switch and devices — assigning LDs to hosts, partitioning bandwidth, and reconfiguring allocations. The FM is the orchestrator of the memory pool.

### Dynamic Memory Allocation and the Stranded-Memory Problem

The economic motivation for pooling is the **stranded-memory problem**. In a conventional server fleet, every server is provisioned with enough memory for its worst-case workload, but at any given moment most servers use far less than their maximum — so a large fraction of the fleet's total DRAM sits idle, "stranded" in servers that don't currently need it while other servers are memory-starved. DRAM is one of the most expensive components in the datacenter, so stranded memory is enormously costly.

CXL memory pooling lets a cloud operator **over-provision aggregate memory and allocate it dynamically**: a pool of CXL Type 3 memory is shared among many hosts, and capacity is assigned to whichever hosts currently need it. A host that needs more memory is granted additional LDs from the pool; a host that no longer needs it releases capacity back. Reallocation is not instantaneous — memory must be scrubbed (to prevent data leakage between tenants), remapped in page tables, and brought online, a process on the order of seconds — so pooling targets capacity-tier elasticity rather than microsecond-scale reallocation. But the payoff is large: reduced total DRAM purchase, less stranded capacity, and the ability to offer elastic memory to workloads whose footprint varies (such as AI inference, where KV-cache size depends on context length and batch).

### CXL Memory Latency and NUMA Awareness

CXL memory is not free of cost: it adds latency relative to local DDR. A local DDR5 access is on the order of **80 nanoseconds**; a CXL 2.0 memory access adds roughly **100–150 nanoseconds** on top, for a total in the 170–250 ns range, while CXL 3.0 improves the additional latency toward 80–100 ns. This is acceptable for a **capacity tier** — memory that holds data not on the hottest path — but not for latency-critical working sets. The implication is that CXL memory is best used as a **second memory tier**, with the hottest data in local DRAM and colder data in CXL.

Software must be **NUMA-aware** to use CXL memory well. CXL-attached memory appears to the operating system as additional **NUMA (Non-Uniform Memory Access) nodes** — but nodes with no local CPU (so-called "CPU-less" or "memory-only" NUMA nodes) and higher access latency. The Linux kernel's NUMA balancing and the emerging **tiered-memory management** subsystems (discussed in the software section) are responsible for placing hot pages in fast local DRAM and migrating cold pages to slower CXL memory, transparently to applications where possible. Getting this tiering right — promoting and demoting pages based on access frequency — is an active area of kernel development and is essential to realizing CXL's promise without degrading application performance.

## CXL 3.0 and 3.1 — Fabric and Peer-to-Peer

CXL 3.0, riding the doubled bandwidth of PCIe 6.0, transforms CXL from a point-to-point-and-single-switch technology into a **fabric** capable of connecting hundreds of hosts and devices at rack scale.

### Multi-Level Switching and Rack-Scale Fabrics

CXL 3.0 supports **multi-level switching** — up to multiple switch levels between a host and a device — enabling a **rack-scale CXL fabric** that connects large numbers of hosts to a large shared memory pool. This is the architectural basis for the disaggregated-memory rack: a chassis of memory devices, a chassis of compute, and a CXL fabric stitching them together so that any host can be allocated memory from any device. The fabric supports far more hosts and devices than the single-switch CXL 2.0 design, moving CXL from "memory expander for one server" toward "memory fabric for a rack."

### Peer-to-Peer Coherence and Device-to-Device

CXL 3.0 introduces **peer-to-peer (P2P) coherent access between devices** without routing through the host CPU. Previously, two accelerators wishing to share data coherently had to go through the host's coherence agent; CXL 3.0 extends CXL.cache to cover **D2D (Device-to-Device)** transactions, so accelerators can coherently access each other's memory directly. This enables a cluster of accelerators sharing a coherent memory space — a capability directly relevant to multi-accelerator AI systems, and a point of overlap (and competition) with NVLink and UALink for accelerator-to-accelerator communication.

### Back-Invalidation and Enhanced Coherence

CXL 3.0 generalizes **back-invalidation (BI)**: the mechanism by which a memory device can invalidate cache lines that a host holds. Back-invalidation is what allows a memory expander or near-memory accelerator to **manage coherence from the device side** — for example, when a computational-memory device modifies data, it can invalidate stale copies in host caches. This device-managed coherence (the HDM-DB model) reduces latency for device-heavy access patterns and is essential for the fabric and P2P capabilities, where coherence must be maintained across many devices without funneling everything through a single host home agent.

### Fabric Addressing

To route coherent transactions across a multi-switch fabric, CXL 3.0 defines a **global addressing scheme**. Devices and ports carry identifiers (a Fabric/Port/Device/LD addressing hierarchy) that allow a transaction to be routed to the correct device and logical device across multiple switch hops. This global addressing is what turns the collection of switches and devices into a coherent fabric rather than a set of isolated links.

## The CXL Competitive Landscape

CXL has attracted an unusually broad set of participants, because it touches CPUs, accelerators, memory, switches, and controllers. The following surveys the major players and their products.

### CPU Vendors

**Intel** is CXL's originator and most committed champion, positioning CXL as the datacenter's universal memory and coherence fabric. **Sapphire Rapids** (4th-gen Xeon Scalable) was the first server CPU to ship CXL (1.1) support; **Emerald Rapids** continued it; and **Granite Rapids** advances toward CXL 2.0. Intel has demonstrated CXL memory-expansion reference designs and integrates CXL into its Agilex FPGAs. Intel's strategic interest is clear: as a memory-and-platform company, CXL strengthens the value of the CPU platform and creates new attach points for Intel silicon.

**AMD** supports CXL across its EPYC line: **Genoa** (4th-gen EPYC) brought CXL 1.1, **Turin** advances toward CXL 2.0, and AMD uses CXL for coherent host-memory access from its MI300-series accelerators. AMD's Pensando DPU portfolio and its broader Infinity Fabric heritage position it to build CXL-attached devices and memory expanders.

**Arm** integrates CXL into its **Neoverse** server cores (N2, V2), and Arm-based server vendors — Ampere, and AWS's Graviton line — adopt CXL. Arm's earlier CCIX effort folded into CXL, and Arm's coherence mesh (CMN) interoperates with CXL at the system level.

### Memory Vendors

The memory makers are racing to build CXL Type 3 devices, because CXL creates an entirely new category of memory product beyond the DIMM:
- **Samsung** introduced **CMM-D (CXL Memory Module — DRAM)**, DRAM-based Type 3 expanders, and **CMM-B**, an LPDDR5-based variant, and has demonstrated CXL memory modules at industry events (Flash Memory Summit). Samsung also pursues **PIM (Processing-in-Memory)** variants that place compute inside the memory device, exposed over CXL — a path toward computational memory.
- **SK Hynix** introduced **CMM-H**, a CXL DRAM module (e.g., 128 GB CXL 2.0 Type 3), and has announced CXL 3.0 modules. SK Hynix's HBM dominance gives it strong incentives to extend its memory leadership into CXL.
- **Micron** has developed CXL memory-expansion products (including CXL-attached devices and a CXL SSD line) and is investing in CXL DRAM. Micron's entry rounds out the "big three" DRAM makers all committing to CXL.

### Controllers, Switches, and IP

Building a CXL memory device requires a **memory controller** that bridges the CXL link to the DRAM, and this has spawned a specialized silicon segment:
- **Montage Technology** builds CXL memory-buffer and controller chips, leveraging its DDR registering-clock-driver heritage.
- **Rambus** offers CXL memory controller IP and chips.
- **Microchip** offers CXL controllers and switch fabric silicon, and integrates CXL into its FPGAs.
- **Astera Labs** offers CXL memory controllers and smart-fabric connectivity, alongside its PCIe/CXL retimer business — a major beneficiary of the CXL buildout.

For switching, **Xconn Technologies** is a notable startup building dedicated **CXL switch silicon** for both CXL 2.0 and 3.0, enabling the memory-pooling and fabric use cases. **Microchip** also addresses CXL switching.

### UALink versus CXL — Complementary or Competitive?

A frequent point of confusion is the relationship between **CXL and UALink (Ultra Accelerator Link)**. UALink, backed by AMD, Intel, Broadcom, Cisco, Google, HPE, Meta, and Microsoft, targets **GPU-to-GPU scale-up** interconnect — high-bandwidth (200 GB/s-class) links connecting accelerators in a scale-up domain, an open alternative to NVIDIA's NVLink. CXL targets **coherent memory and CPU-device** interconnect. In the cleanest framing, they are **complementary**: CXL for memory expansion, pooling, and CPU-accelerator coherence; UALink for the high-bandwidth accelerator mesh. In practice, there is overlap — CXL 3.0's peer-to-peer device coherence touches the accelerator-to-accelerator space that UALink also addresses — and the two standards will jostle at that boundary. But the broad industry consensus treats CXL as the memory/coherence fabric and UALink as the scale-up accelerator fabric, with both serving as open counterweights to NVIDIA's proprietary stack.

## The CXL Software Stack

Hardware is only half the story; CXL memory is useless without operating-system and runtime support to discover it, manage it, and place data on it intelligently.

### Linux Kernel Support

The **Linux kernel CXL subsystem** (under `drivers/cxl/`) was merged starting around kernel 5.12 and has matured steadily. It includes drivers for device enumeration (`cxl_pci`), for Type 3 memory (`cxl_mem`), and the machinery to create **CXL regions** — contiguous host-physical-address ranges backed by one or more CXL memory devices — exposed and configured via sysfs. The kernel integrates CXL memory as NUMA nodes and provides the hooks for tiered-memory management. Driver and subsystem development remains active, tracking the evolving 2.0/3.0 features (pooling, dynamic capacity, fabric management).

### Tiered Memory Management

Because CXL memory is a slower tier, the kernel must decide which pages live in fast local DRAM and which in slower CXL memory, and migrate pages between tiers as access patterns change. The relevant technologies include:
- **DAMON (Data Access MONitor)**, a kernel subsystem that monitors memory-access patterns efficiently and can drive page promotion/demotion decisions (via DAMOS, the DAMON-based Operating Schemes).
- **Tiering libraries and tools** such as **Memkind** and `numactl`, which let applications and administrators direct allocations to specific memory tiers.
- The conceptual analogy to Intel's earlier **Optane Persistent Memory** Memory Mode (transparent caching of DRAM over PMEM) versus App Direct mode (explicit application control) maps onto CXL's transparent-tiering versus explicit-placement choices.

The goal is **transparent tiering**: applications run unmodified while the kernel keeps hot data in fast memory and cold data in CXL memory, capturing most of the capacity and cost benefit with minimal performance loss. Achieving this robustly across diverse workloads is one of the central software challenges of the CXL era.

### Emulation and Testing

Because CXL hardware is new and expensive, **QEMU CXL emulation** is invaluable: QEMU can emulate CXL switches and memory devices, letting operating-system, firmware, and management-software developers test CXL enumeration, region creation, and pooling without physical hardware. This has accelerated the maturation of the software stack ahead of broad hardware availability.

### Error Handling and RAS

Memory is failure-prone, and CXL must integrate into the platform's **RAS (Reliability, Availability, Serviceability)** machinery. CXL errors are reported through **CPER (Common Platform Error Records)** and the platform error-handling flows, and CXL memory supports **poison handling** — flagging memory media errors as "poison" so that consuming the corrupted data triggers a controlled error rather than silent corruption. Robust RAS is essential for CXL memory to be trusted in production, especially in pooled configurations where a device failure could affect multiple hosts; isolating faults so that one device's failure does not cascade is a key requirement of the fabric manager and the platform.

## Extended Deep Dive: The Latency Tax and Why Tiering Is Hard

The central practical challenge of CXL memory is the **latency tax** and the software complexity of managing it, and this deserves elaboration because it determines whether CXL delivers its promise in practice. Local DDR5 access is ~80 ns; CXL 2.0 memory adds ~100–150 ns on top (the CXL protocol processing, the link traversal, and the device's own memory access), for a total in the 170–250 ns range. This is not catastrophic — it is comparable to accessing memory on a remote NUMA socket in a multi-socket server — but it is enough that placing a workload's *hot* data in CXL memory degrades performance noticeably. CXL memory is therefore a **second tier**: a place for warm and cold data, with hot data kept in local DRAM.

The hard part is deciding, dynamically and transparently, which data is hot and which is cold, and migrating pages between tiers accordingly. This is the **tiered-memory management** problem, and it is genuinely difficult. The kernel must monitor access patterns at fine enough granularity to identify hot pages (the DAMON subsystem, File 04 above) without the monitoring itself consuming excessive resources; it must migrate hot pages up to fast memory and cold pages down to CXL, paying the migration cost (copying the page, updating page tables, flushing TLBs) only when the benefit justifies it; and it must avoid pathological thrashing (repeatedly migrating a page that is accessed intermittently). Different workloads have wildly different access patterns — a database, a graph analytic, an AI inference server with a large KV cache — and a tiering policy that works for one may fail another. Getting this right across diverse workloads, transparently to applications, is an active and unfinished area of kernel development, and it is the principal reason CXL memory's real-world benefit has lagged its theoretical promise. The hardware is ready before the software, a recurring pattern in memory-system innovation (the same was true of NUMA and of persistent memory), and the maturation of tiered-memory management is the gating factor for CXL's broad adoption as a capacity tier.

## Extended Deep Dive: Memory Pooling Economics in Detail

The economic case for CXL memory pooling — attacking stranded memory — rewards quantification, because it is the clearest near-term justification for CXL infrastructure. In a conventional fleet, each server is provisioned for its peak memory need, but measurements at hyperscalers consistently show that a large fraction of provisioned DRAM is idle at any moment: a server running a memory-light workload uses a fraction of its DIMMs while, elsewhere in the fleet, a memory-heavy workload is constrained. Because DRAM is one of the largest line items in server cost (often rivaling or exceeding the CPU), this stranded memory represents a major capital inefficiency — potentially tens of percent of the fleet's total DRAM sitting idle.

CXL pooling lets an operator provision a **shared pool** of CXL memory across many servers and allocate it dynamically (File 04), so the aggregate provisioning can be closer to the *aggregate* demand rather than the sum of *per-server peaks*. The savings depend on the statistical smoothing across many workloads (the more uncorrelated the per-server demands, the more the peaks cancel and the smaller the pool needed), and published analyses suggest meaningful single-digit-to-low-double-digit percentage reductions in total DRAM for realistic fleets — a large absolute saving at hyperscale. The costs to weigh against this are the CXL latency tax (the pooled memory is a slower tier), the CXL infrastructure (switches, controllers, the fabric), and the operational complexity of the fabric manager allocating and scrubbing memory (with reallocation on the order of seconds, suitable for capacity elasticity but not microsecond-scale sharing). The net economic case is workload- and fleet-dependent, but for large operators with diverse workloads and expensive DRAM, memory pooling is the CXL use case with the clearest and most immediate return — which is why the memory makers (Samsung, SK Hynix, Micron) and the controller/switch vendors (Montage, Astera, Xconn) are investing so heavily in Type 3 devices and CXL switching.

## Extended Deep Dive: CXL Versus NVLink-C2C and the Coherence Landscape

It is illuminating to position CXL within the broader landscape of coherent interconnects, particularly against NVIDIA's NVLink-C2C (File 05). Both provide cache-coherent CPU-accelerator memory access, but they embody opposite philosophies. **NVLink-C2C** is proprietary, maximally optimized for NVIDIA's Grace-Hopper/Grace-Blackwell superchips, delivering 900 GB/s of coherent bandwidth — far beyond what a CXL link over PCIe lanes provides — at the cost of being a closed, single-vendor solution usable only within NVIDIA's integrated products. **CXL** is an open, multi-vendor standard, usable by any CPU, accelerator, or memory device, delivering lower bandwidth (bounded by the PCIe lanes it runs over) but with the enormous advantage of interoperability and a broad ecosystem. The trade-off mirrors the InfiniBand-versus-Ethernet (File 07) and NVLink-versus-UALink (File 24) tensions that recur throughout this database: proprietary integration and peak performance versus open standardization and ecosystem breadth.

The likely resolution is a **division of roles**: NVLink-C2C (and NVLink generally) for the highest-bandwidth, tightly integrated CPU-GPU and GPU-GPU coherence within NVIDIA's products; CXL for the open, multi-vendor memory-expansion, pooling, and CPU-device-coherence use cases across the broader industry; and UALink for the open scale-up accelerator mesh. CXL's role is less about competing with NVLink on raw bandwidth than about being the **universal, open memory and coherence fabric** for the rest of the datacenter — the standard that lets any CPU attach pooled memory, any accelerator coherently access host memory, and (eventually, over optics) any host reach a building-scale memory pool. In a landscape of proprietary high-performance fabrics, CXL is the open standard that knits the heterogeneous, multi-vendor datacenter together, which is precisely why it attracted the industry-wide consolidation that subsumed Gen-Z, CCIX, and OpenCAPI.

## Extended Deep Dive: Cache Coherence Fundamentals and Why CXL.cache Is Hard

To appreciate what CXL.cache accomplishes, one must understand the cache-coherence problem it solves, and it is worth developing because coherence is among the hardest problems in computer architecture. When multiple agents (CPU cores, accelerators) cache copies of the same memory location, the system must ensure they all see a consistent value — if one agent writes, the others' stale copies must be invalidated or updated. CPUs solve this within a socket with hardware coherence protocols (MESI and its variants: each cache line is in a Modified, Exclusive, Shared, or Invalid state, and agents snoop each other's caches to maintain consistency). Extending this coherence across a link to an external accelerator — so the accelerator can cache host memory and the host can cache accelerator memory, all kept consistent — is what CXL.cache does, and it is genuinely difficult because the coherence messages (snoops, invalidations, ownership requests) must traverse the link with bounded latency and without deadlock, while preserving the memory-ordering guarantees that software relies on.

The **bias modes** (host bias and device bias, File 04 above) are CXL's clever optimization to manage coherence overhead: rather than incurring a coherence transaction on every access, software can declare a memory region as device-biased (the accelerator owns it, accesses it locally without involving the host) or host-biased (the host coordinates), flipping the bias as the access pattern shifts. The **back-invalidation** mechanism (HDM-DB, File 04) lets a memory device participate actively in coherence, invalidating host copies when needed — essential for the device-managed coherence of fabrics. The reason CXL succeeded where earlier coherent-interconnect efforts (CCIX, OpenCAPI, Gen-Z) fragmented is partly that it got this coherence model right and broadly acceptable, and partly the industry's exhaustion with fragmentation — but the technical achievement of a clean, deadlock-free, performant cross-link coherence protocol, layered on PCIe, is substantial and is what makes the heterogeneous, coherent, disaggregated datacenter possible.

## Extended Deep Dive: CXL and the Reshaping of Server Architecture

The deeper implication of CXL is that it reshapes the **fundamental architecture of the server**, and tracing this clarifies why CXL is more than a memory expander. The classic server is a fixed bundle: a CPU with a fixed number of DIMM slots, some PCIe devices, all in one box, with memory capacity and bandwidth permanently tied to the CPU socket. CXL begins to dissolve this. With CXL memory expansion, memory capacity is no longer bounded by the socket's DIMM slots — a server can attach far more memory over CXL. With CXL pooling, memory is no longer owned by a single server but drawn from a shared pool and allocated dynamically. With CXL 3.0 fabrics and peer-to-peer, accelerators can coherently share memory across the rack. The endpoint is the **disaggregated, composable server**: pools of compute, memory, and accelerators, connected by CXL (and, for the highest bandwidth, NVLink/UALink) fabrics, composed on demand into the right machine for each workload (File 18, File 24).

This is a profound architectural shift, and it is why the industry — CPU vendors, memory makers, hyperscalers — invested so heavily in CXL despite its early-stage challenges (the latency tax, the immature tiering software). The prize is the end of stranded resources (memory allocated where it is idle while needed elsewhere) and the flexibility of composing machines from pools rather than buying fixed servers — a more efficient, more flexible datacenter. The full realization is years away (it requires mature pooling, fabric management, tiering software, and eventually CXL over optics to reach beyond the rack, File 24), but the direction is set, and CXL is the interconnect that makes memory disaggregation — the hardest and most valuable form of disaggregation, because memory is so latency-sensitive — possible. In reshaping how memory attaches to compute, CXL is reshaping the server itself, and through it, the datacenter.

## Conclusion: CXL as the Memory Fabric of the Disaggregated Datacenter

CXL began as a coherence interconnect and is becoming the memory fabric of the datacenter. Its trajectory — from a point-to-point expander (1.1), to a pooled, switched resource (2.0), to a rack-scale coherent fabric with peer-to-peer device communication (3.0/3.1) — traces the broader arc of datacenter architecture toward **disaggregation and composability**: pools of compute, memory, and accelerators connected by fast coherent fabrics and composed on demand into the right machine for each workload. The memory wall that motivated CXL is not going away — if anything, AI's appetite for memory capacity and bandwidth is intensifying it — and CXL is the industry's coordinated, consolidated answer. Its eventual extension over optical links (CXL over fiber, File 24) would push coherent memory pools beyond the rack to building scale, completing the flattening of the interconnect hierarchy that File 01 described. CXL's success is not guaranteed — latency, software maturity, and the economics of pooling all present real challenges — but no other technology is as well-positioned to redefine what "memory" means in the datacenter. The next chapter descends one more level, to the die-to-die interconnects (UCIe, HBM, and the chiplet ecosystem) on which the accelerators and CPUs at the heart of all this are themselves built.
