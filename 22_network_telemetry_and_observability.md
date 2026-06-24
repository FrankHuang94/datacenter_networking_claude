# Network Telemetry, Observability, and AIOps

## Introduction: You Cannot Operate What You Cannot See

A datacenter network spanning hundreds of thousands of ports, carrying the synchronized bursts of AI collectives and the long tail of cloud services, cannot be operated by intuition. It requires **observability** — the ability to measure, in fine-grained detail and near-real-time, what every link, queue, and flow is doing — so that operators (and increasingly, automated systems) can detect congestion, diagnose failures, and optimize performance. The legacy model of polling devices for counters every few minutes is hopelessly inadequate for a fabric where a microburst lasting microseconds can stall a billion-dollar training run. This chapter covers the modern telemetry and observability stack — in-band telemetry, streaming telemetry, flow sampling, BGP monitoring, AIOps, and eBPF-based observability — that gives operators the visibility AI-era networks demand. It builds on the gNMI/OpenConfig foundations of File 16 and the congestion-control mechanisms of Files 06 and 17.

## In-band Network Telemetry (INT)

**In-band Network Telemetry (INT)** embeds telemetry metadata directly into live data packets as they traverse the network. As a packet passes through each switch, the switch (typically a P4-programmable ASIC, File 14) appends metadata — its switch ID, the egress queue depth at that moment, a hop timestamp, link utilization — into an INT header stack within the packet. The receiver (or a monitoring endpoint) reads the accumulated INT stack, reconstructing the **exact path the packet took and the precise congestion state at every hop**, at microsecond granularity. This is far richer than any sampled or polled telemetry: it reveals the actual queueing experienced by real traffic.

INT is the basis of **Alibaba's HPCC** congestion control (File 06), which uses the precise queue-depth information to compute exact rate adjustments. INT comes in modes — **INT-MD (metadata, full per-hop insertion)** and **INT-XD (export from each node)** — and the Intel Tofino reference implementation popularized it. INT's cost is the per-packet overhead and the need for programmable switches, but for diagnosing the transient congestion that plagues AI fabrics, its visibility is unmatched.

## Streaming Telemetry

**Streaming telemetry** replaces the request-response polling of SNMP with a **push** model: devices stream their state continuously to collectors. The standard mechanism is **gNMI Subscribe** (File 16), with modes for one-time snapshots (ONCE), periodic polling (POLL), and continuous streaming (STREAM). Devices push per-port counters, buffer occupancy, error rates, BGP state, and more at intervals that have shrunk from seconds toward **100 ms and even 10 ms**. The telemetry pipeline typically comprises:
- A **message bus** (Kafka) to ingest the high-volume stream.
- **Time-series databases** (InfluxDB, TimescaleDB, Prometheus) to store it.
- **Visualization** (Grafana) for the network operations center.
- **Alerting and analytics** layered on top.

This streaming model is essential for AI fabrics, where the operator must observe buffer occupancy and congestion events at sub-second granularity to correlate network behavior with collective-communication stalls and to feed congestion-control tuning (File 17).

## sFlow and IPFIX — Flow-Level Visibility

Complementing per-device telemetry, **flow-level** monitoring reveals traffic patterns:
- **sFlow** exports **sampled packets** (1 in N) from switches — a low-overhead, hardware-supported mechanism present in virtually every switch, giving a statistical view of traffic composition and enabling anomaly and DDoS detection.
- **IPFIX (RFC 7011)**, the standardized successor to Cisco's NetFlow v9, exports **flow records** (summaries of each flow: addresses, ports, byte/packet counts, timestamps) for traffic analysis, capacity planning, and security analytics.

Both are widely used for understanding *what* traffic is flowing where, complementing the *how-congested* view of INT and streaming telemetry.

## BGP Monitoring Protocol

**BMP (BGP Monitoring Protocol, RFC 7854)** exports a router's BGP RIB (Routing Information Base) and route updates to a monitoring station, enabling **BGP analytics**: tracking route changes, detecting anomalies and hijacks (File 19), analyzing path selection, and auditing the control plane. Supported by Cisco IOS-XR, Junos, FRRouting, and others, BMP gives operators visibility into the routing control plane that complements the data-plane telemetry from INT and streaming.

## AIOps for Networking

**AIOps** applies machine learning to network operations, evolving through increasing sophistication:
- **Threshold-based alerting** (the legacy: alert if a counter exceeds a fixed value) — simple but noisy and brittle.
- **Statistical anomaly detection** (e.g., isolation forests) that learn normal behavior and flag deviations, reducing false positives.
- **Time-series forecasting** (LSTM and other models) that predict traffic and capacity needs, enabling proactive action.
- **Automated root-cause analysis** that correlates events across the fabric to pinpoint the source of a problem.

Commercial AIOps for networking includes **Juniper Mist** (AI-driven wired and wireless operations), **Cisco DNA Assurance**, and others, while hyperscalers build bespoke ML pipelines on their telemetry streams. For AI fabrics, AIOps holds particular promise: detecting the subtle congestion and tail-latency patterns that degrade training efficiency, and automating the diagnosis and remediation that would otherwise require expert human analysis of overwhelming telemetry volumes.

## eBPF-Based Observability

The host side of observability has been transformed by **eBPF** (File 20). Tools like **Pixie** (open-source) and **Cilium Hubble** use eBPF to observe network and application behavior **in the Linux kernel without code changes or application instrumentation** — capturing per-packet latency, parsing Layer 7 protocols (HTTP, gRPC, MySQL, Kafka) on the fly, and providing service-level visibility into the actual traffic. This kernel-level, zero-instrumentation observability is central to cloud-native operations, giving developers and operators deep visibility into microservice communication without modifying applications, and complementing the network-device telemetry with host-level and application-level context.

## Conclusion

Observability is the prerequisite for operating modern networks, and the stack has been transformed from minute-granularity SNMP polling to a rich, real-time ecosystem: in-band telemetry revealing exact per-hop congestion in live packets, streaming telemetry pushing device state at sub-second granularity, flow sampling (sFlow/IPFIX) showing traffic composition, BMP exposing the routing control plane, AIOps applying machine learning to detect and diagnose problems, and eBPF delivering zero-instrumentation host and application visibility. For AI fabrics, where transient microbursts and tail-latency events directly determine the efficiency of enormously expensive training runs, this fine-grained, real-time observability — feeding both human operators and automated congestion-control tuning — is not a luxury but a necessity. You cannot operate what you cannot see, and the modern telemetry stack is how the AI-era network is made visible. The next chapter steps back to survey the vendor landscape — the companies whose products and competition constitute the industry this database describes.
