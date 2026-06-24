# Building Lossless RoCEv2 AI Fabrics — Architecture, Tuning, and Operations

## Introduction: The Operational Craft

The preceding chapters established the theory: RDMA needs losslessness (File 08), Ethernet achieves losslessness through PFC and congestion control (File 06), and AI training stresses the fabric with synchronized collective bursts (File 15). This chapter is about the **practice** — the operational craft of designing, tuning, and operating a production RoCEv2 fabric that delivers InfiniBand-approaching performance for AI training without succumbing to the pathologies (PFC storms, deadlocks, ECMP imbalance, microbursts) that lurk in every lossless Ethernet network. It is the most hands-on chapter in the database, and the difficulty of getting these details right is, candidly, one of InfiniBand's enduring competitive advantages (File 07). It also covers the InfiniBand operational equivalents (Subnet Manager, SHARP configuration) for contrast.

## End-to-End RoCEv2 Fabric Design

A production AI RoCE fabric is designed top to bottom for non-blocking, lossless, low-latency collective communication:
- **Server NICs**: RoCEv2-capable NICs with hardware congestion control and GPUDirect RDMA — NVIDIA **ConnectX-7** (400G), Marvell **FastLinQ**, or Intel **E810**-class adapters — one or more per GPU, providing the per-GPU scale-out bandwidth (400–800 Gbps).
- **Top-of-rack (leaf) switches**: high-radix, low-latency switches with adequate buffering and robust PFC/ECN support — e.g., Arista 7050X/7060X, Cisco Nexus 9300, Juniper QFX5220 — typically built on Broadcom Trident/Tomahawk or equivalents.
- **Spine switches**: high-bandwidth switches (Arista 7800R, Cisco Nexus 9500, or fixed 51.2T platforms) providing non-blocking bisection.
- **Fabric links**: 400G/800G DR4/FR4 optics for inter-switch links, with a **non-blocking (1:1 oversubscription)** Clos for the AI fabric so that no collective is bandwidth-starved (File 06).
- **Topology**: often a **rail-optimized** design (File 15), mapping each GPU's NIC to a specific rail/plane so that collectives traverse minimal hops and avoid cross-rail congestion.

The design philosophy: provision so generously (non-blocking, deep enough buffers) and tune so carefully (congestion control keeping queues short) that PFC rarely fires and the fabric behaves nearly losslessly under the worst-case synchronized burst.

## PFC Domain Design

**Priority Flow Control** is the lossless foundation, and designing the PFC domain correctly is the first operational task:
- **Traffic-class isolation**: place RDMA traffic on a dedicated priority (commonly 802.1p priority 3, sometimes 4), and enable PFC **only** for that class. Other traffic (TCP management, storage, etc.) runs lossy on separate classes, so PFC backpressure on RDMA never stalls unrelated traffic and vice versa.
- **Single lossless class to avoid deadlock**: PFC deadlock arises from **cyclic buffer dependencies** — if the buffer-occupancy dependency graph contains a cycle, PFC backpressure can circle indefinitely and freeze the fabric. The simplest robust defense is to use a **single lossless priority class** with a deadlock-free routing/topology (a Clos with up/down forwarding has no cycles), avoiding the multi-class dependency cycles that cause deadlock.
- **PFC watchdog**: enable a **PFC watchdog timer** that detects a port stuck in a paused state for too long (a sign of a PFC storm or deadlock) and takes corrective action (draining the queue, raising an alarm), preventing a local stuck condition from cascading into a fabric-wide stall.

PFC must be treated as a **last-resort safety net**, not the primary congestion mechanism — the goal is for congestion control to keep queues short enough that PFC almost never triggers.

## ECN Marking Configuration

The primary congestion mechanism is **ECN-based DCQCN** (File 06), and configuring the switches' ECN marking is critical:
- Switches mark ECN using a **WRED (Weighted Random Early Detection)** profile on the RDMA queue, defined by a **minimum threshold** (below which no marking), a **maximum threshold** (above which all packets are marked), and a **marking probability** that ramps between them.
- Typical AI-fabric settings begin marking at a moderate buffer occupancy (e.g., **~20–30%** of the queue) so that DCQCN reacts *before* the buffer fills enough to trigger PFC, with the PFC threshold set higher (e.g., ~80%) as the backstop. The separation between the ECN-marking threshold (lower) and the PFC threshold (higher) is what lets graceful ECN-driven rate reduction act first and PFC act only if ECN fails to control the burst in time.
- **Buffer sizing**: a port must buffer enough to absorb a burst during the control loop's reaction time — roughly (RTT × line rate) of data. At 400G with ~100 µs of round-trip control latency, that is on the order of several MB per port, informing the choice between shallow-buffer (Tomahawk) and deep-buffer (Jericho) switches for the fabric (File 14).

## DCQCN Parameter Tuning

**DCQCN** has numerous tunable parameters on the NIC, and tuning them for the cluster's scale and message-size distribution is a specialized skill:
- **Rate-reduction timer (Rp_timer, ~55 ms)** and **byte-reset counter (~150 KB)** govern how the sender recovers its rate after a congestion event.
- **Rate-increase timer (~5 ms)** and the **additive-increase (AI)** and **hyperactive-increase (HAI)** parameters govern how quickly the sender ramps back up.
- **Alpha update frequency** governs how the sender's congestion estimate (alpha) tracks the marked-packet fraction.
- **Minimum rate (e.g., 10% of line rate)** prevents the sender from collapsing to zero.

These defaults are starting points; the optimal values depend on cluster size (more endpoints → more potential synchronized senders), message-size distribution (large AllReduce tensors versus small tensor-parallel collectives), and topology. Mis-tuned DCQCN either reacts too slowly (letting PFC fire and congestion spread) or too aggressively (under-utilizing the fabric). Operators iterate on these parameters using fabric telemetry and collective-benchmark results.

## Routing Design for RDMA: The ECMP Imbalance Problem

A subtle but devastating problem in RoCE fabrics is **ECMP hash imbalance**, the "elephant flow" problem. ECMP load-balances flows across the Clos paths by hashing the packet's 5-tuple. But RDMA bulk transfers between a given pair of endpoints typically use a **single queue pair**, hence a single 5-tuple, so **all** the data of a large transfer hashes to **one path** — and when many such elephant flows collide on the same link while leaving others idle, the fabric is congested despite having ample aggregate bandwidth. Solutions:
- **Increased entropy**: vary the UDP source port per QP or use multiple QPs per flow to spread a transfer across paths (requires the transport to tolerate the resulting reordering).
- **Adaptive routing**: switches (e.g., NVIDIA Quantum for IB, or Ethernet switches with adaptive routing / Spectrum-X) dynamically reroute to less-congested paths.
- **Flowlet switching**: split a flow into "flowlets" (bursts separated by gaps) and rebalance flowlets across paths, exploiting natural gaps to avoid reordering within a burst.
- **Packet spraying** (per-packet load balancing across all paths), as advocated by the Ultra Ethernet Consortium, with a reordering-tolerant transport at the endpoints — the most thorough solution, requiring transport-layer support.

ECMP imbalance is one of the most common causes of disappointing real-world RoCE performance, and addressing it is a central goal of next-generation Ethernet transports.

## Monitoring and Operations

Operating a RoCE fabric requires watching the right signals:
- **RDMA NIC counters**: **out_of_buffer** (the receiver was not ready — backpressure/credit exhaustion events), **out_of_sequence** (packets arriving reordered — a sign of multipath reordering or loss), and **duplicate_requests** (retransmissions — a sign of loss/timeout). Rising values on these counters are early warnings of fabric trouble.
- **PFC counters and storm detection**: monitor PFC pause frames sent/received per priority; a sustained high rate indicates congestion that ECN is failing to control, and a stuck condition indicates a potential storm/deadlock.
- **Buffer-occupancy telemetry**: stream queue depths (via gNMI/INT, Files 16/22) to spot microbursts and hot links.
- **Latency-percentile monitoring**: track P99 and P999 latency, not just averages — collectives are barriers, so tail latency determines training throughput. Targets are typically sub-100 µs for training collectives and even tighter for storage.

The operational goal is to detect and remediate congestion hot spots, PFC events, and tail-latency regressions before they degrade training efficiency — and to feed these observations back into topology, routing, and DCQCN tuning.

## InfiniBand Operations for Contrast

For contrast, operating an **InfiniBand** fabric centers on different machinery (File 07):
- **Subnet Manager (OpenSM or vendor SM)** configuration: ensuring a redundant SM is running, managing the LID space, and selecting the **routing algorithm** (DFSSSP or up/down for fat-trees, MINHOP for other topologies).
- **Adaptive routing** enable/disable and tuning on Quantum switches.
- **SHARP tree configuration**: setting up the in-network reduction trees for collective acceleration.
- **Partitioning (PKeys)**: configuring InfiniBand partitions for multi-tenant isolation, the IB equivalent of VLAN/tenant segmentation.

The notable contrast is that InfiniBand delivers losslessness and low latency with **far less tuning** — its credit-based flow control is inherently lossless and deterministic, requiring none of the delicate PFC/ECN/DCQCN balancing that RoCE demands. This operational simplicity is a real (if often under-acknowledged) part of InfiniBand's value proposition, and closing this operational gap — making Ethernet "just work" for AI as InfiniBand does — is precisely the goal of NVIDIA's Spectrum-X and the Ultra Ethernet Consortium.

## Extended Deep Dive: The Three-Layer Defense Against Loss

It is worth synthesizing the lossless-fabric machinery into a single coherent model: a **three-layer defense against packet loss**, each layer acting at a different timescale and as a backstop for the one before. The first and fastest layer is the **physical-layer FEC** (File 02), which corrects the bit errors caused by channel noise so that corruption-induced loss essentially never reaches the transport — operating continuously, per codeword, at nanosecond timescale. The second layer is **congestion control** (DCQCN, HPCC, or the Ultra Ethernet transport), which operates at the round-trip-time timescale (microseconds), sensing incipient congestion via ECN marks or telemetry and reducing senders' rates to keep queues short *before* buffers fill — this is the primary, graceful defense against congestion loss. The third and last-resort layer is **PFC**, which operates at the link timescale when a buffer is about to overflow despite congestion control, pausing the upstream sender to prevent the drop — the safety net that must rarely fire.

The art of fabric tuning, as this chapter has detailed, is arranging these three layers so each does its job and the next only acts when the previous is overwhelmed: strong-enough FEC that corruption loss never occurs; congestion control tuned (ECN thresholds, DCQCN parameters) to keep queues short enough that PFC rarely triggers; and PFC configured (thresholds above the ECN marking point, watchdogs, deadlock-free routing) as a safe backstop. When all three layers are correctly co-designed, the fabric is effectively lossless under the worst-case synchronized burst, congestion is handled gracefully by rate control, and PFC's pathologies almost never manifest. When they are mis-tuned — FEC too weak, ECN thresholds wrong, DCQCN too slow, PFC thresholds overlapping the ECN point — the fabric either drops RDMA packets (the loss catastrophe, File 08) or spreads PFC backpressure into a stall. This three-layer model is the unifying mental framework for everything in lossless-fabric engineering.

## Extended Deep Dive: Benchmarking and Validating a Fabric

Before a RoCE fabric carries production training, and continuously thereafter, it must be **benchmarked and validated**, and the methodology is part of the operational craft. The foundational tools are the **perftest** suite (ib_write_bw, ib_read_bw, ib_send_lat, etc.) that measure raw RDMA bandwidth and latency between pairs of nodes, establishing that the point-to-point fabric performs to spec. Above this, **collective benchmarks** — the NCCL tests (all_reduce_perf, all_to_all_perf) — measure the actual collective bandwidth and latency across many GPUs, which is what training performance ultimately depends on; these reveal whether the fabric delivers its theoretical bisection bandwidth under real collective patterns, or whether ECMP imbalance, congestion, or a slow link is degrading it. Operators run these at increasing scale (8 GPUs, a rack, a pod, the full cluster) to validate that performance scales as the topology promises.

The validation must also probe the **failure and stress** behavior: injecting load to verify that congestion control keeps queues short and PFC rarely fires; failing links to verify deadlock-free rerouting; and running long-duration tests to surface intermittent "gray failures" (File 15) that short tests miss. The telemetry counters (out_of_buffer, out_of_sequence, PFC pause counts) are watched throughout to confirm the fabric is healthy under load. This benchmarking discipline — establishing a performance baseline, validating it scales, and continuously monitoring for regression — is what separates a fabric that reliably delivers its rated performance to expensive GPU fleets from one whose subtle misconfigurations silently waste a fraction of the cluster's value every day. At hyperscale, where that fraction translates to millions of dollars, fabric benchmarking and validation is a high-value, continuous engineering function, not a one-time acceptance test.

## Conclusion

Building a lossless RoCEv2 AI fabric is an exercise in co-designing every layer — NIC, switch, topology, routing, PFC, ECN, and congestion control — so that the fabric absorbs the synchronized bursts of large-scale collective communication without dropping a packet or letting PFC backpressure spread into a stall. The craft lies in the details: isolating the RDMA traffic class, separating ECN and PFC thresholds so graceful control acts before the safety net, sizing buffers for the burst, tuning DCQCN to the cluster, defeating ECMP elephant-flow imbalance with adaptive routing or packet spraying, and monitoring the NIC counters and tail latencies that reveal the fabric's health. Done well, RoCE approaches InfiniBand's performance over an open, multi-vendor, cost-competitive fabric; done poorly, it collapses under the very workload it was built for. The gap in operational simplicity between RoCE and InfiniBand is the gap that the next generation of Ethernet — Ultra Ethernet and Spectrum-X — is racing to close, and the resolution of that race will shape the AI-fabric market for years (Files 15, 23, 24). The next chapter turns to a related fabric discipline: storage networking, where RDMA and NVMe-oF deliver remote storage at near-local latency.
