# AI/ML and HPC Networking — Fabrics, Collectives, and the Scale-Up/Scale-Out Architecture

## Introduction: The Network Is the Computer

In distributed AI training, the network is not an accessory to the computation — it *is* part of the computation. A large language model trained across tens of thousands of GPUs spends a substantial fraction of every training step moving data: synchronizing gradients, exchanging activations, routing tokens to experts. When the network stalls, the GPUs — the most expensive resource in the datacenter — sit idle. The efficiency of an AI cluster, measured as the fraction of peak FLOPS actually delivered to useful training, is determined as much by the network as by the accelerators. This chapter brings together the threads of the preceding chapters — Ethernet (File 06), InfiniBand (File 07), RDMA and collectives (File 08), NVLink and the scale-up fabrics — into the architecture of complete AI and HPC fabrics. It covers the communication patterns of distributed training, the scale-up versus scale-out distinction, the networking architectures of NVIDIA, AMD, Intel, Google, and Meta, the collective-communication algorithms in detail, and a comparative analysis of the AI-fabric options. It is the synthesis toward which much of the database builds.

## AI Training Networking Requirements

### Communication Patterns in Distributed Training

Distributed deep-learning training parallelizes the work across many accelerators, and each form of parallelism imposes a distinct communication pattern:
- **Data parallelism**: each GPU holds a full copy of the model and processes a different slice of the training batch. After each forward-backward pass, the GPUs must **synchronize gradients** — every GPU's gradient must be summed with every other's (an **AllReduce**) before the optimizer step. Data parallelism's communication is the AllReduce of the full gradient, once per step.
- **Tensor (model) parallelism**: a single layer's computation is split across GPUs (e.g., a matrix multiply partitioned across devices). This requires frequent **AllReduce/AllGather** within the layer's computation — many small, latency-sensitive collectives per layer, demanding very high bandwidth and low latency, which is why tensor parallelism is kept within a high-bandwidth scale-up domain (NVLink).
- **Pipeline parallelism**: the model's layers are split into stages across GPUs, with activations passed from stage to stage (point-to-point sends), and micro-batches pipelined to keep stages busy. Communication is the activation hand-off between adjacent stages — less frequent but bandwidth-significant.
- **Expert parallelism (Mixture-of-Experts, MoE)**: each GPU hosts different "expert" sub-networks, and tokens are routed to the appropriate experts via an **AllToAll** exchange — every GPU sends different data to every other. AllToAll is the most demanding pattern, and MoE models can require enormous all-to-all bandwidth.
- **Sequence parallelism**: long input sequences are split across the sequence dimension, adding further collectives.

Real frontier-model training combines several of these (3D or 4D parallelism), and the network must serve all their patterns simultaneously. The mapping of parallelism dimensions onto the physical topology — tensor parallelism within the NVLink domain, data parallelism across the scale-out fabric — is one of the central optimizations of large-scale training.

### The Communication-to-Compute Ratio

The pressure on the network has intensified because **compute has scaled faster than interconnect bandwidth**. Each GPU generation roughly multiplies FLOPS by ~2.5×, while interconnect bandwidth roughly doubles — so the ratio of compute to communication tightens, making the network an ever-larger fraction of the bottleneck. Concretely, gradient AllReduce for a model of P parameters moves on the order of the model size per step; for GPT-3-scale (175B parameters) this is tens of gigabytes synchronized per step, demanding tens of GB/s per GPU sustained; for MoE models at the trillion-parameter scale, AllToAll expert routing can demand on the order of a terabyte per second per GPU at full utilization. The network bandwidth per GPU (hundreds of gigabits to ~1 terabit per second) must keep the GPU's collectives from dominating step time.

### Collective Bandwidth and the Roofline

The key collectives and their traffic costs:
- **AllReduce**: total traffic ≈ 2 × (N−1)/N × (data size) — the factor of ~2 reflecting the reduce-scatter then all-gather phases of the ring algorithm; bandwidth efficiency approaches (N−1)/N → ~100% for large N.
- **AllGather / ReduceScatter**: each moves ≈ (N−1)/N × (full data size); these are the two halves of a ring AllReduce.
- **AllToAll**: each GPU sends a distinct shard to every other; worst-case traffic scales with N × shard size — the most bandwidth-intensive.

A **roofline** analysis frames whether a training job is compute-bound or communication-bound: the arithmetic intensity (FLOPs per byte communicated) of the workload, compared to the ratio of the accelerator's FLOPS to its interconnect bandwidth, determines which resource binds. For example, synchronizing a 70B-parameter model's gradients across 8 GPUs at 900 GB/s NVLink takes on the order of tens of milliseconds — a meaningful fraction of the ~100 ms compute time per step — so even at NVLink bandwidth, communication is a first-order term, and at the lower bandwidths of the scale-out fabric, it dominates without careful overlap and in-network reduction.

## Scale-Up versus Scale-Out

A foundational concept in AI networking is the distinction between **scale-up** and **scale-out** fabrics:

- **Scale-up fabric** connects accelerators *within a tightly coupled domain* — a server or a rack-scale unit — with ultra-high bandwidth, very low latency, and often cache coherence or a shared memory space. This is **NVLink/NVSwitch** (NVIDIA), **Infinity Fabric** (AMD within MI300X), **Xe Link** (Intel), and the emerging open **UALink**. Bandwidth: 900 GB/s (NVLink 4.0) to 1.8 TB/s (NVLink 5.0) per GPU; latency under a microsecond. Scale-up serves the frequent, small, latency-sensitive collectives of **tensor parallelism**.

- **Scale-out fabric** connects the scale-up domains to each other across the cluster — server to server, rack to rack — over **Ethernet (RoCEv2)** or **InfiniBand**. Bandwidth: 400 Gbps per GPU (NDR IB or 8×50G RoCE) to 800 Gbps (XDR IB or 800GbE); latency 600 ns (IB) to a few microseconds (RoCE). Scale-out serves the less frequent, larger, bandwidth-sensitive collectives of **data and pipeline parallelism**.

The **scale-up domain** size is a critical architectural parameter: the more GPUs that can be connected at NVLink-class bandwidth, the more tensor parallelism (and the larger the models) that can be served at full bandwidth. NVIDIA's NVLink Switch connects 8 GPUs in a DGX H100; the **Blackwell GB200 NVL72** extends the scale-up domain to **72 GPUs** in a single rack at 1.8 TB/s per GPU — a dramatic expansion. AMD and the UALink consortium aim to build large open scale-up domains as an alternative. Both fabrics are necessary because no single technology economically provides both NVLink's intra-domain bandwidth and Ethernet/IB's cluster-wide reach.

## NVIDIA Networking Architecture for AI

NVIDIA's vertically integrated AI fabric is the reference against which all others are measured:

- **DGX H100 system**: 8 H100 GPUs interconnected by **NVLink Switch (3rd Gen)** in a full 900 GB/s all-to-all mesh (the scale-up domain), plus **4 ConnectX-7 NICs** providing 4×400G NDR InfiniBand (1.6 Tbps) to the scale-out fabric. Within the box, GPUs talk over NVLink; between boxes, over InfiniBand.
- **DGX SuperPOD**: 32 DGX H100 systems connected through **Quantum-2 (NDR) InfiniBand** leaf and spine switches in a non-blocking fat-tree, with **SHARP** performing in-network gradient aggregation. The SuperPOD is the reference architecture for thousands-of-GPU training.
- **Blackwell GB200 NVL72**: a rack-scale unit of **36 Grace CPUs + 72 Blackwell GPUs** connected by **NVLink Switch (4th Gen)** at **1.8 TB/s bidirectional per GPU** — 72 GPUs sharing on the order of 130 TB/s of aggregate NVLink bandwidth within the rack — with scale-out via InfiniBand NDR/XDR or Ethernet. The NVL72 dramatically enlarges the scale-up domain, enabling far larger models to be served at NVLink bandwidth.
- **Magnum IO software stack**: **NCCL** (the collective library, with NVLS/NVLink-SHARP and IB-SHARP support), **GPUDirect RDMA** (the NIC DMAs directly to/from GPU memory, bypassing the CPU and host memory), **UCX**, CUDA-aware MPI, and **DOCA** (the BlueField DPU framework). This software is as much a moat as the hardware — NCCL's deep optimization for NVIDIA topologies is a key reason NVIDIA fabrics deliver superior collective performance.

## AMD AI Networking

**AMD's MI300X** is a chiplet marvel (8 GPU dies + 4 CPU/IO dies via 3D stacking and CoWoS, with 8 HBM3 stacks at 5.3 TB/s, File 05), but its networking philosophy contrasts sharply with NVIDIA's: AMD uses **standard Ethernet/RoCEv2 for inter-node** communication (around 7×100 GbE RDMA ports, ~700 Gbps per GPU for scale-out) and has **no proprietary NVLink equivalent** in current shipping products — relying on Infinity Fabric within the package and Ethernet between nodes. AMD's **RCCL** (ROCm Collective Communications Library) provides collectives. For scale-up, AMD is a leading backer of the open **UALink** standard (File 24), targeting large open scale-up domains for future generations (MI350X and beyond). AMD's argument: a standard Ethernet fabric is more flexible, lower cost, and customer-operable than NVIDIA's proprietary NVLink, and open standards (UALink, Ultra Ethernet) will close the performance gap. NVIDIA's counter: NVLink's bandwidth is irreplaceable for the largest models' tensor parallelism. This is the central strategic contest of AI networking.

## Intel Gaudi and AI Networking

**Intel's Gaudi** accelerators take a distinctive **Ethernet-unified** approach: Gaudi 3 integrates **24×200 GbE (or 24×100 GbE in prior gens) RDMA ports directly on the chip**, using RoCEv2 for *both* scale-up (intra-node, via on-board Ethernet) and scale-out (inter-node) — a single, unified Ethernet fabric with no proprietary scale-up interconnect. This radically simplifies the network model (everything is RoCEv2 Ethernet) and aligns with the open-Ethernet thesis. An 8-Gaudi server provides enormous aggregate Ethernet bandwidth, and Intel pairs Gaudi with merchant switches (e.g., Marvell Teralynx) in non-blocking fat-trees. Intel's collective library (part of the Habana/SynapseAI stack) and its embrace of Ultra Ethernet position Gaudi as the most thoroughly Ethernet-native of the major AI accelerators.

## Google TPU Networking

**Google's TPU** fabric is the most architecturally distinctive, eschewing both Ethernet and InfiniBand for a proprietary interconnect:
- **TPU v4**: 4,096 TPU chips per pod, interconnected in a **3D torus** via the **ICI (Inter-Chip Interconnect)** — 6 optical links per chip (two in each of x, y, z dimensions) — with **optical circuit switching (Palomar)** to reconfigure the torus topology dynamically. Each TPU has ~600 Gbps of ICI bandwidth across its 6 links.
- **Palomar OCS**: MEMS-based optical circuit switches (e.g., 128×128) reconfigure the inter-block topology in ~milliseconds, letting Google match the topology to the workload's communication pattern — different collectives prefer different topologies — and enabling fault tolerance and incremental upgrades (File 11).
- **TPU v5e / v5p**: v5e cost-optimized for inference, v5p for training (pods of thousands of chips); v5p provides high per-chip ICI bandwidth balanced to its compute. Google's ICI + OCS approach gives it topological flexibility that static Clos fabrics lack — a key differentiator and a glimpse of the reconfigurable-fabric future (File 24).

## Meta AI Networking

**Meta** takes a pragmatic, heterogeneous approach:
- **Research SuperCluster (RSC)**: an early large GPU cluster (hundreds of DGX A100 nodes) on a non-blocking **200G RoCEv2 Ethernet** fabric.
- **Grand Teton (H100)**: Meta's Open-Compute H100 platform, using NVIDIA ConnectX-7 **InfiniBand NDR** for its largest training clusters, while using Ethernet for inference and other workloads.
- **Scale**: Meta has announced AI infrastructure on the order of **350,000 H100 GPUs**, deployed across multiple large clusters, and has publicly documented running large-scale training (e.g., Llama models) over both InfiniBand and RoCE fabrics — using its experience to push the open-Ethernet (Ultra Ethernet) agenda. Meta's willingness to build very large RoCE fabrics, and to publish its findings, makes it a key force in proving Ethernet viable for frontier-scale AI.

## Collective Communication Algorithms in Depth

The performance of an AI fabric is realized (or squandered) by the collective-communication algorithms:

- **Ring AllReduce**: N nodes in a logical ring; each repeatedly sends a chunk to its neighbor while reducing received chunks; after 2(N−1) steps every node has the full reduced result. Bandwidth efficiency approaches optimal ((N−1)/N of link bandwidth is useful), making it ideal for **large messages** (bandwidth-bound), but its latency grows linearly with N — a problem at very large scale. NCCL's default for large tensors.
- **Recursive Halving/Doubling (butterfly/hypercube)**: completes in log₂(N) rounds, each communicating with a partner at increasing distance; latency grows only logarithmically with N, making it superior for **small messages** (latency-bound) — relevant to the small collectives of tensor parallelism.
- **Tree AllReduce**: reduces up a tree and broadcasts down; logarithmic latency; used where latency matters.
- **NVLS (NVLink SHARP)**: the NVLink Switch performs the reduction **in-switch** as data flows through it, so AllReduce within an NVLink domain effectively achieves N× the per-link bandwidth (the data need not traverse N−1 hops); NCCL uses NVLS for NVLink topologies — a major within-node AllReduce accelerator.
- **SHARP (InfiniBand)**: the Quantum switch performs FP16/BF16/INT reduction **in-network** across nodes, so multi-node AllReduce traffic does not traverse links repeatedly — roughly doubling effective bandwidth versus ring and eliminating the O(N) ring traversal (File 07). NCCL uses IB-SHARP for multi-node training.

In-network reduction (NVLS, SHARP) is the key to scaling collectives beyond the point where ring algorithms' linear latency becomes prohibitive — and replicating this capability in open Ethernet (via the Ultra Ethernet Consortium and in-network-computing efforts) is a central goal of the open-fabric camp.

## AI Fabric Comparison

| Fabric | BW/GPU | Latency | Lossless mechanism | Collective accel. | Scope | Open/proprietary |
|---|---|---|---|---|---|---|
| NVLink 4.0 | 900 GB/s | <1 µs | Credit (link) | NVLS | Scale-up | Proprietary |
| NVLink 5.0 (Blackwell) | 1.8 TB/s | <1 µs | Credit | NVLS | Scale-up | Proprietary |
| InfiniBand NDR | 400 Gbps | ~600 ns | Credit-based | SHARP | Scale-out | Proprietary (NVIDIA) |
| InfiniBand XDR | 800 Gbps | ~600 ns | Credit-based | SHARP | Scale-out | Proprietary (NVIDIA) |
| RoCEv2 (400/800G) | 400–800 Gbps | 1–3 µs | PFC + DCQCN/UEC | (UEC in-net, emerging) | Scale-out | Open (multi-vendor) |
| Spectrum-X | 400–800 Gbps | ~1–2 µs | PFC + NVIDIA CC | Spectrum in-net | Scale-out | NVIDIA Ethernet |
| UALink 1.0 | ~200 GB/s/link | <1 µs | Credit | (planned) | Scale-up | Open |
| Google ICI (TPU v5p) | ~hundreds GB/s | low | Credit | in-fabric | Scale-up+out | Proprietary (Google) |

This table crystallizes the landscape: NVLink dominates scale-up bandwidth (proprietary); InfiniBand leads scale-out latency and has mature in-network reduction (proprietary, NVIDIA); RoCEv2/Ultra Ethernet offers openness and cost at the price of harder congestion management; Spectrum-X is NVIDIA's Ethernet hedge; UALink is the open scale-up challenger; and Google's ICI is a proprietary, reconfigurable alternative entirely.

## Conclusion

AI and HPC networking is where every layer of this database converges: the die-to-die HBM that feeds the GPU, the NVLink scale-up fabric that binds GPUs within a domain, the InfiniBand or RoCE scale-out fabric that connects domains into clusters, the optical links and circuit switches that carry it, the switch silicon that forwards it, and the collective-communication algorithms and in-network reduction that orchestrate it. The architecture is fundamentally two-tier — a high-bandwidth scale-up domain for tensor parallelism, a cluster-wide scale-out fabric for data and pipeline parallelism — and the central strategic contest is between NVIDIA's vertically integrated, proprietary stack (NVLink + InfiniBand/Spectrum-X + NCCL) and the open-standards challenge (Ethernet/Ultra Ethernet + UALink) led by AMD, Intel, and the hyperscalers, with Google's proprietary TPU fabric as a third path. The network is the computer: the efficiency of the world's largest AI clusters is decided by how well these fabrics deliver the synchronized, lossless, low-latency collective communication that training demands. The chapters that follow examine the supporting disciplines — network operating systems and programmability (File 16), the operational craft of building lossless RoCE fabrics (File 17), storage networking (File 18) — and ultimately the vendor landscape (File 23) and future roadmaps (File 24) that will decide how this contest resolves.
