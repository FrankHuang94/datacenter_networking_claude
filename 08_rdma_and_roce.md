# RDMA, RoCE, iWARP, and High-Performance Transport Protocols

## Introduction: Moving Data Without the CPU

Remote Direct Memory Access (RDMA) is the technology that makes high-performance distributed computing possible. Its premise is deceptively simple: let one computer read or write the memory of another computer directly, without involving the remote CPU, without copying data through kernel buffers, and without the latency and overhead of the traditional sockets-and-kernel networking stack. The consequences are profound — sub-microsecond message latency, near-zero CPU overhead for data movement, and the ability to scale collective communication across thousands of accelerators. RDMA is the transport beneath InfiniBand (File 07), beneath the Ethernet AI fabrics (File 06), and beneath modern storage (File 18). This chapter examines RDMA itself, its three Ethernet/IP incarnations (RoCE, iWARP, Soft-RoCE), the programming model, the collective-communication libraries built atop it, and the operational realities of deploying RDMA fabrics.

## RDMA Fundamentals

The classic networking stack — application, sockets, TCP/IP, kernel, NIC — imposes three costs that RDMA eliminates. First, **CPU involvement**: every packet traverses the kernel, consuming CPU cycles for protocol processing, interrupt handling, and context switches. Second, **memory copies**: data is copied from application buffers to kernel socket buffers to NIC buffers (and the reverse on receive), consuming memory bandwidth. Third, **latency**: the journey through the kernel stack adds microseconds. For a web server these costs are tolerable; for a 10,000-GPU training job exchanging gradients every few milliseconds, they are fatal.

RDMA eliminates all three through three mechanisms:
- **Kernel bypass**: the application communicates directly with the NIC from user space, via memory-mapped queues, with no system calls on the data path.
- **Zero-copy**: the NIC DMAs data directly between application memory and the wire, with no intermediate copies.
- **One-sided operations**: RDMA Read and RDMA Write transfer data to/from remote memory without the remote CPU participating at all — the remote NIC handles the transfer autonomously.

The RDMA programming interface is the **verbs API**, exposed in Linux via **libibverbs**. The cost RDMA imposes in exchange is **memory registration**: before the NIC can DMA to or from a buffer, that buffer must be registered (pinned in physical memory and given access keys), an operation with non-trivial overhead that well-designed applications amortize through registration caching. The RDMA model originated with InfiniBand and was standardized broadly by the RDMA Consortium; its semantics now span InfiniBand, Ethernet, and emerging optical fabrics.

## RoCE — RDMA over Converged Ethernet

**RoCE** brings InfiniBand's RDMA transport to Ethernet, allowing organizations to run RDMA over their existing Ethernet infrastructure rather than deploying a separate InfiniBand network. It exists in two versions:

- **RoCEv1** encapsulates the InfiniBand transport directly in an Ethernet frame (EtherType 0x8915). It operates only at Layer 2 — within a single broadcast domain / subnet — because it has no IP header and therefore cannot be routed. This limited its usefulness in large fabrics.

- **RoCEv2**, the version used everywhere today, encapsulates the InfiniBand transport inside a **UDP/IP** packet (destination UDP port **4791**). Because it has an IP header, RoCEv2 is **routable** across a Layer 3 Clos fabric, making it suitable for large datacenter networks. A RoCEv2 packet carries the IP and UDP headers, then InfiniBand's **GRH (Global Routing Header)** and **BTH (Base Transport Header)**, then the RDMA payload. The InfiniBand transport semantics — Reliable Connection queue pairs, RDMA Read/Write/Atomic, the same verbs API — operate unchanged over the Ethernet physical and IP network layers.

RoCEv2's promise is "InfiniBand RDMA on Ethernet hardware," but it inherits Ethernet's best-effort nature, so it requires the lossless machinery (PFC) and congestion control (DCQCN, HPCC, etc.) of File 06 to perform well. The UDP source port is deliberately varied per flow to provide entropy for the fabric's ECMP load balancing — though, as File 17 discusses, RDMA's tendency to use a single queue pair (and thus a single 5-tuple) for bulk transfers undermines this, creating the "elephant flow" load-imbalance problem.

## iWARP — RDMA over TCP

**iWARP (internet Wide Area RDMA Protocol)** takes a different approach: it layers RDMA semantics on top of **TCP** (via the MPA, DDP, and RDMAP protocol layers). Because it runs over TCP, iWARP inherits TCP's reliability and congestion control, so it does **not** require a lossless fabric (no PFC needed) and works over ordinary routed networks, including the wide-area internet. The trade-off is **higher latency** than RoCE — TCP processing, even when offloaded to the NIC, adds overhead — and a more complex NIC implementation. iWARP found adoption in storage and some enterprise contexts (Chelsio and Intel were notable iWARP NIC vendors; Marvell's FastLinQ supports it), but it lost the AI/HPC fabric battle to RoCEv2 and InfiniBand, which deliver lower latency. iWARP's principal appeal — robustness over lossy, routed networks without special fabric configuration — remains relevant where a lossless fabric is impractical.

## Soft-RoCE and Software RDMA

**Soft-RoCE (rxe)** is a software implementation of RoCE in the Linux kernel that runs over an ordinary Ethernet NIC, without RDMA hardware. It implements the RoCE protocol in software, so it has none of the performance benefits of true RDMA (the CPU does the work, with copies and kernel involvement), but it lets developers write and test RDMA applications, and lets systems interoperate with RDMA peers, without dedicated hardware. Soft-RoCE is invaluable for development, continuous integration, and education, and it provides a fallback path for nodes lacking RDMA NICs.

## The RDMA Programming Model

Programming RDMA directly involves a specific sequence of operations:
1. **Memory registration**: the application registers buffers, pinning them and obtaining a **local key (lkey)** for local access and a **remote key (rkey)** that it shares with peers to authorize their RDMA access to those buffers.
2. **Queue pair creation and connection**: the application creates queue pairs and establishes connections (for RC), exchanging the necessary addressing and key information out of band.
3. **Posting work requests**: the application posts Work Requests (WRs) — RDMA Write, RDMA Read, Send, Receive, Atomic — to the send or receive queue.
4. **Completion polling**: the application polls the Completion Queue to detect when operations finish, avoiding interrupt latency.

Writing directly to verbs is powerful but laborious, so most applications use higher-level abstractions. **libfabric (the OpenFabrics Interfaces, OFI)** is a portable abstraction layer over verbs and other RDMA/networking providers, presenting a consistent API that can target InfiniBand verbs, RoCE, iWARP, shared memory, or vendor-specific transports — insulating applications from the underlying fabric. **UCX (Unified Communication X)** is another widely used abstraction (below).

## MPI and Collective Communication over RDMA

The dominant programming model for HPC and much of AI distributed computing is **MPI (Message Passing Interface)**, implemented by **OpenMPI**, **MVAPICH2**, and others. MPI runs over RDMA through transport layers such as **UCX (Unified Communication X)**, which automatically selects the best available transport — InfiniBand verbs, RoCEv2, shared memory for intra-node, or others — based on the topology, and which is used not only by OpenMPI but by distributed frameworks including Spark and TensorFlow. UCX abstracts the messy details of transport selection and optimization, presenting a clean point-to-point and one-sided communication API.

The operations that matter most for AI and HPC are **collective communications** — operations involving all (or a group of) participants:
- **AllReduce**: every participant contributes a value (e.g., a gradient tensor), the values are combined (typically summed), and every participant receives the result. This is the dominant operation in data-parallel training (gradient synchronization).
- **AllGather**: every participant's data is gathered and distributed to all (used in tensor-parallel and sharded-optimizer schemes).
- **ReduceScatter**: the reduction is computed and the result scattered in pieces to participants — the second half of a ring AllReduce.
- **AllToAll**: every participant sends distinct data to every other (the defining operation of Mixture-of-Experts routing).
- **Broadcast** and **Reduce**: one-to-all and all-to-one.

The efficiency of these collectives depends on the algorithm. **Ring AllReduce** achieves near-optimal bandwidth efficiency (each participant's link carries roughly the minimum necessary data) and is preferred for large messages; **tree** and **recursive-halving/doubling** algorithms achieve lower latency for small messages. The libraries that implement these for accelerators are critical: **NCCL (NVIDIA Collective Communications Library)** is the dominant one for NVIDIA GPUs, implementing ring, tree, and **NVLS (NVLink SHARP)** algorithms and exploiting NVLink, NVSwitch, and InfiniBand SHARP for in-network reduction; **RCCL** is AMD's equivalent for ROCm; **oneCCL** is Intel's. These libraries, and the collective algorithms they embody, are examined in depth in File 15.

## RDMA for Storage: NVMe-oF

RDMA is transformative for storage as well as compute. **NVMe-oF (NVMe over Fabrics)** carries the NVMe storage protocol's submission/completion queue model over a network fabric, and **NVMe/RDMA** maps NVMe queues directly onto RDMA queue pairs, achieving remote-storage latencies that approach local NVMe. The latency comparison is stark: **NVMe/RDMA over RoCEv2** achieves under ~20 µs for a 4 KB read; **NVMe-oF over InfiniBand** can be under ~10 µs; while **NVMe/TCP** (NVMe over ordinary TCP, no RDMA) is around ~100 µs. RDMA's order-of-magnitude latency advantage for remote storage is what makes storage disaggregation — separating compute from storage capacity across a fast fabric — practical without crippling performance. File 18 covers storage networking in full.

## Congestion and the RDMA Loss Catastrophe

The defining operational challenge of RDMA over Ethernet is that **packet loss is catastrophic** for the Reliable Connection transport. When a packet is dropped, the RC transport must retransmit — and in the basic go-back-N scheme, it retransmits from the last acknowledged sequence number, potentially re-sending a large amount of data and destroying throughput. (Modern NICs implement selective retransmission and improved loss recovery, but loss remains far more expensive than for TCP, which was designed around it.) This is why RoCEv2 fabrics must be **lossless**: a single congestion drop can cripple a flow.

Losslessness is achieved with **PFC**, but PFC's pathologies (head-of-line blocking, congestion spreading, deadlock — File 06) mean PFC must be a last resort. The primary defense is **congestion control** that keeps queues short enough that PFC rarely fires: **DCQCN** (ECN-based), **HPCC** (telemetry-based), **Timely/Swift** (delay-based). The interaction of FEC (preventing corruption loss), PFC (preventing congestion loss), and congestion control (preventing PFC from firing) is the delicate co-design that File 02 introduced and File 17 operationalizes.

## RoCEv2 Deployment Best Practices

Deploying a production RoCEv2 fabric well requires attention to numerous details (elaborated in File 17):
- **Traffic isolation**: place RDMA traffic on a dedicated priority/VLAN with PFC enabled for that class only, isolating it from lossy best-effort traffic.
- **ECN marking thresholds**: configure switch WRED/ECN to begin marking at a moderate buffer occupancy (e.g., 20–30%) so DCQCN reacts before buffers fill and PFC triggers.
- **DCQCN parameter tuning**: tune the rate-reduction timer, byte-reset counter, rate-increase parameters, alpha-update frequency, minimum rate, and additive/hyperactive increase to the cluster's scale and message-size distribution.
- **PFC safety**: enable PFC watchdogs to detect and break stuck-PFC conditions, and design topology and routing to be free of cyclic buffer dependencies (deadlock-free).
- **Monitoring**: watch the RDMA NIC counters that reveal fabric health — **out_of_buffer** (backpressure/receiver-not-ready events), **out_of_sequence** (reordering), and **duplicate_requests** (retransmission indicators) — and alert on PFC storms and rising tail latency.

These practices are what separate a RoCEv2 fabric that approaches InfiniBand performance from one that collapses under the synchronized bursts of large-scale training. The difficulty of getting them right at scale is, candidly, one of InfiniBand's enduring advantages — InfiniBand delivers losslessness and low latency with far less tuning — and it is the gap that the Ultra Ethernet Consortium and NVIDIA's Spectrum-X are each trying to close.

## Conclusion

RDMA is the quiet foundation of high-performance distributed computing: kernel bypass, zero-copy, one-sided memory access, delivering the sub-microsecond latency and near-zero CPU overhead that distributed training and disaggregated storage require. Its three Ethernet incarnations — RoCEv2 (the performance leader, requiring a lossless fabric), iWARP (robust over lossy networks, but slower), and Soft-RoCE (software, for development) — bring RDMA to the ubiquitous Ethernet world, while InfiniBand provides it natively. Above RDMA sit the abstraction layers (libfabric, UCX) and the collective libraries (NCCL, RCCL, oneCCL) that turn raw memory-to-memory transfer into the AllReduce and AllToAll operations that define AI training. And beneath it all lies the unforgiving requirement of losslessness, the delicate multi-layer co-design that makes RDMA over Ethernet work — the subject to which File 17 is devoted. RDMA is where the abstract promise of "the network" meets the concrete reality of moving petabytes of gradients between tens of thousands of GPUs without dropping a packet or wasting a CPU cycle.
