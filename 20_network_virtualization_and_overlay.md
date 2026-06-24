# Network Virtualization, Overlay Networks, and Cloud Networking

## Introduction: The Network as Software Abstraction

Network virtualization decouples the logical network that workloads see from the physical network that carries their packets, exactly as server virtualization decoupled the virtual machine from the physical server. This decoupling is what makes the multi-tenant cloud possible: thousands of tenants each get their own private, isolated virtual network — their own address space, their own segmentation, their own policies — all riding over a shared physical fabric that knows nothing of tenants. The mechanism is the **overlay**: encapsulating tenant packets inside outer packets that the physical underlay routes, so the underlay sees only the encapsulation and the tenants see only their virtual network. This chapter covers the overlay encapsulations (VXLAN, GENEVE, GRE), their control planes (BGP EVPN), the source-routing and label-switching alternatives (SRv6, MPLS), the platform implementations (VMware NSX), and the high-performance software dataplanes (eBPF/XDP) and container networking (Kubernetes CNI) that make virtualization fast. It builds on the protocol stack of File 02 and the security of File 19.

## VXLAN — The Dominant Datacenter Overlay

**VXLAN (Virtual Extensible LAN, RFC 7348)** is the workhorse datacenter overlay. It encapsulates a complete Layer 2 Ethernet frame inside an outer **UDP/IP** packet (outer UDP destination port **4789**), with an 8-byte VXLAN header carrying a **24-bit VNI (VXLAN Network Identifier)** — 16 million virtual segments, versus the 4,096 of VLANs. The encapsulation/decapsulation endpoint is the **VTEP (VXLAN Tunnel Endpoint)**, which can reside in a hypervisor virtual switch, in a NIC (hardware offload), or in a physical switch ASIC (Broadcom Trident/Tomahawk and NVIDIA hardware VTEPs). The total overhead (~50 bytes) makes hardware offload and jumbo frames important.

A critical detail is **entropy for load balancing**: the VTEP sets the **outer UDP source port** as a hash of the inner flow, so that the underlay's ECMP (File 02) spreads different inner flows across different physical paths — the same entropy mechanism whose failure causes the elephant-flow imbalance of RoCE fabrics (File 17). VXLAN thus lets the underlay load-balance tenant traffic without understanding it.

### BGP EVPN — The VXLAN Control Plane

Early VXLAN relied on multicast or flood-and-learn to discover which VTEP hosted which MAC — inefficient and unscalable. The modern control plane is **BGP EVPN (Ethernet VPN, RFC 7432)**, which uses BGP (specifically MP-BGP with the EVPN address family) to **distribute MAC and IP reachability** among VTEPs: each VTEP advertises the MACs and IPs behind it, so other VTEPs learn the mapping without flooding. EVPN also enables **multi-homing** (a host connected to multiple leaf switches via Ethernet Segment Identifiers, ESI, for redundancy and load sharing) and **symmetric IRB (Integrated Routing and Bridging)** for efficient inter-subnet routing within the overlay. BGP EVPN over a VXLAN data plane is the standard modern datacenter overlay architecture, and it underlies the virtual networks of AWS VPC, Azure VNet, and GCP VPC (each with proprietary enhancements).

## GENEVE — The Extensible Overlay

**GENEVE (Generic Network Virtualization Encapsulation, RFC 8926)** generalizes VXLAN with a **variable-length, extensible header** carrying type-length-value (TLV) options, designed so that SDN controllers can attach arbitrary metadata to tunneled packets (e.g., for service chaining, security context, or telemetry). GENEVE's flexibility makes it the encapsulation of choice for software-defined platforms — **VMware NSX-T** uses GENEVE, as does Juniper Contrail — and **Open vSwitch** supports it natively. The trade-off versus VXLAN is that GENEVE's variable-length header is slightly more complex to offload in hardware, though modern NICs increasingly support it.

## SRv6 and MPLS — Source Routing and Label Switching

For traffic engineering and WAN/VPN services, two other paradigms matter:
- **Segment Routing over IPv6 (SRv6)** encodes the path a packet should take as a list of "segments" (IPv6 addresses) in a **Segment Routing Header (SRH)**, so the source (or ingress) determines the path and the core nodes simply follow it — **no per-flow state in the core**, unlike traditional MPLS traffic engineering. SRv6 supports traffic engineering, VPNs, and service-function chaining, and is deployed by Alibaba, SoftBank, and others for WAN traffic engineering; **compressed SRv6 (C-SRv6 / uSID)** reduces the header overhead.
- **MPLS (Multi-Protocol Label Switching)** uses short labels (rather than IP lookups) to forward packets along pre-established label-switched paths, underpinning carrier **L3VPN (RFC 4364)** and traffic engineering for two decades. New deployments increasingly use **Segment Routing MPLS (SR-MPLS)** — the MPLS forwarding plane with the stateless SR control plane — or migrate to SRv6, but the enormous installed base keeps MPLS central to carrier and inter-datacenter networks.

## VMware NSX and Platform Network Virtualization

**VMware NSX** is the leading commercial network-virtualization platform, implementing in software the entire suite of network services — logical switching (overlay), logical routing, distributed firewall (micro-segmentation, File 19), load balancing, and VPN — as a layer atop any physical fabric. **NSX-T** uses GENEVE encapsulation with VTEPs in the ESXi hypervisor kernel (or as N-VDS virtual switches), enforcing the **distributed firewall** at each VM's virtual NIC so that east-west traffic is inspected and segmented **without hairpinning** to a physical firewall. NSX exemplifies the "network as software" model: the physical fabric provides simple IP transport, and all the rich network services live in software at the hypervisor edge, programmable and mobile with the workloads.

## eBPF and High-Performance Software Dataplanes

The performance of software networking has been transformed by **eBPF (extended Berkeley Packet Filter)** and **XDP (eXpress Data Path)**, which allow sandboxed programs to run in the Linux kernel's networking fast path — processing packets at or near line rate, in the kernel, without a custom kernel module and without copying packets to user space. Use cases:
- **Cilium**: a Kubernetes networking and security plugin (CNI) built on eBPF, providing identity-based network policy, load balancing, and observability at high performance, increasingly the default for cloud-native networking.
- **Cloudflare** uses eBPF/XDP for line-rate DDoS mitigation, dropping attack packets in the kernel before they consume resources.
- **Hubble** (Cilium's observability layer) and **Pixie** provide eBPF-based observability (File 22).

eBPF lets operators implement custom, high-performance per-flow policy and observability in software, on commodity hardware, bridging the gap between the flexibility of software and the speed once requiring dedicated hardware — a profound shift in how datacenter networking functions are built.

## Kubernetes and Container Networking

The rise of containers and Kubernetes created a new layer of network virtualization: the **Container Network Interface (CNI)**, the standard for plugging networking into Kubernetes pods. CNI plugins embody different philosophies:
- **Calico**: a BGP-based, often overlay-free approach that routes pod IPs natively, scaling well and integrating with the underlay's BGP.
- **Flannel**: a simple VXLAN overlay for pod-to-pod connectivity.
- **Cilium**: the eBPF-based plugin (above), providing high-performance networking, identity-aware policy, and observability.
- **Multus**: enables multiple network interfaces per pod, important for workloads needing both a management network and a high-performance data network.

For high-performance and AI workloads, container networking integrates with the hardware: **SR-IOV** (File 03) gives a container direct access to a NIC virtual function for near-native performance; **DPDK**-accelerated CNIs bypass the kernel for maximum throughput; and **RDMA in Kubernetes** — via the NVIDIA Network Operator and GPU Operator, which expose RDMA devices and configure RoCE/InfiniBand for pods — brings GPUDirect RDMA and high-performance collective communication to containerized AI training. This convergence of container networking with hardware acceleration is essential for running AI workloads on Kubernetes at scale.

## Conclusion

Network virtualization is the abstraction that makes the multi-tenant cloud possible, decoupling the logical networks that workloads see from the physical fabric that carries them. The overlay — VXLAN with its BGP EVPN control plane, the extensible GENEVE, and the source-routing and label-switching alternatives SRv6 and MPLS — encapsulates tenant traffic so that the underlay can route it without understanding it, while platforms like VMware NSX implement the full suite of network services in software at the hypervisor edge. The performance gap that once made software networking slow has been closed by eBPF/XDP (line-rate kernel programmability) and by hardware acceleration (SR-IOV, DPDK, RDMA, offload to DPUs), and the container era has produced a rich CNI ecosystem that, for AI workloads, integrates directly with RDMA and GPUDirect. The result is a datacenter network that is simultaneously a simple, high-bandwidth physical transport and an infinitely flexible, software-defined, multi-tenant abstraction — the physical and the virtual co-designed, as every layer of this database has shown them to be. The next chapter returns to hardware, examining the optical-transceiver market — the 400G, 800G, and 1.6T modules — whose technology and fierce competition underpin all the fabrics described throughout.
