# Storage Networking — FC, NVMe-oF, and Distributed Storage Fabrics

## Introduction: Feeding the Compute

Storage networking is the discipline of connecting compute to data across a fabric, and it has been transformed twice over: first by the shift from spinning disks to flash (which made the network, not the media, the bottleneck), and again by the rise of RDMA (which let remote storage approach local-storage latency). In the AI datacenter, storage networking feeds the data pipelines that supply training (streaming petabytes of training data to GPUs) and absorbs the checkpoints that protect against failure (writing terabytes of model state periodically). This chapter covers the storage-networking protocols — Fibre Channel, iSCSI, and the modern NVMe-over-Fabrics family — and the distributed and disaggregated storage architectures built on them. It builds on the RDMA foundations of File 08 and connects to the disaggregation themes of Files 04 and 24.

## Fibre Channel — The Enterprise Storage Incumbent

**Fibre Channel (FC)** has been the dominant enterprise storage-area-network (SAN) protocol for over two decades, prized for its reliability, deterministic performance, and mature management in mission-critical enterprise environments. FC is a purpose-built, lossless, low-latency fabric for block storage, with its own physical layer, framing, and addressing — entirely separate from Ethernet/IP.

FC has scaled through generations: **Gen 5 (16GFC)**, **Gen 6 (32GFC)**, **Gen 7 (64GFC, ~2019)**, and **Gen 8 (128GFC, ~2022)**. The FC **fabric** is switched (the old arbitrated-loop topology is obsolete), with hosts and storage logging into the fabric (**FLOGI**) and to each other (**PLOGI**), and **zoning** controlling which initiators may access which targets — the SAN equivalent of network segmentation. The market is a duopoly: **Brocade (now part of Broadcom)** and **Cisco (MDS series)** dominate FC switching.

FC's extensions and decline-and-persistence are instructive. **FCoE (Fibre Channel over Ethernet)** attempted to converge FC onto Ethernet (using DCB/PFC for losslessness) but largely **failed to achieve broad adoption** — the operational complexity of converging two very different networks outweighed the benefit, and most operators kept FC and Ethernet separate or moved to IP storage. **FC-NVMe** carries NVMe commands over the FC fabric, modernizing FC for flash. FC persists strongly in enterprise storage arrays (NetApp, Dell EMC, HPE, Pure Storage) where its reliability and the installed base sustain it, even as new cloud and AI deployments favor Ethernet-based storage networking.

## iSCSI — Block Storage Over TCP/IP

**iSCSI** carries SCSI block-storage commands over ordinary TCP/IP, requiring no special fabric — any Ethernet network and commodity NIC will do. This simplicity made iSCSI hugely popular in small/medium enterprises and virtualization environments (VMware iSCSI datastores are ubiquitous). Performance options range from a pure **software initiator** (the host CPU does all the iSCSI/TCP work) to **iSCSI HBAs** (hardware offload of the iSCSI and TCP processing). iSCSI's latency — on the order of 100–200 µs, dominated by TCP processing — is acceptable for general virtualization but not for the most demanding workloads, where NVMe-oF over RDMA is far faster. iSCSI's enduring appeal is its zero-special-hardware simplicity and its use of the existing IP network.

## NVMe-over-Fabrics Architecture

**NVMe (Non-Volatile Memory Express)**, the protocol designed for PCIe-attached flash (File 03), revolutionized storage with its massively parallel queue model (up to 65,535 I/O queues). **NVMe-over-Fabrics (NVMe-oF)** extends this model across a network, carrying NVMe's submission/completion queue semantics over a fabric transport so that remote flash can be accessed with the same low-overhead, highly parallel model as local NVMe. The NVMe controller model — namespaces, submission and completion queues, the streamlined command set — is preserved across the fabric, with an **NVMe initiator** (host) and **NVMe target** (storage controller or flash device).

NVMe-oF defines several **transports**:
- **NVMe/RDMA**: maps NVMe queues directly onto RDMA queue pairs (over RoCEv2 or InfiniBand), achieving the lowest latency — **under ~20 µs for a 4 KB read over RoCEv2, under ~10 µs over InfiniBand** — by exploiting RDMA's kernel bypass and zero-copy (File 08). This is the transport of choice for high-performance all-flash arrays.
- **NVMe/TCP**: carries NVMe over ordinary TCP, requiring no RDMA hardware, at the cost of higher latency (**~100 µs**, TCP-dominated). NVMe/TCP is gaining rapid adoption (VMware, Red Hat, Microsoft Azure Disk) precisely because it needs no special fabric, bringing NVMe-oF to commodity networks.
- **NVMe/FC**: carries NVMe over Fibre Channel, modernizing FC SANs for flash.

The **latency comparison** is the crux: NVMe/RDMA (~10–20 µs) approaches local NVMe and makes storage disaggregation viable; NVMe/TCP (~100 µs) trades latency for fabric simplicity; iSCSI (~100–200 µs) is the legacy baseline. The right choice depends on whether the workload needs RDMA-class latency (databases, high-performance analytics, AI data pipelines) or can tolerate TCP-class latency for operational simplicity.

High-performance NVMe-oF targets are often built with the **SPDK (Storage Performance Development Kit)** — Intel's user-space, polled-mode storage stack that bypasses the kernel for maximum IOPS and minimum latency, the storage analog of RDMA's kernel bypass.

## Distributed Object and Block Storage

Beyond the SAN model, **distributed software-defined storage** spreads data across many commodity servers connected by the Ethernet fabric:
- **Ceph** is the leading open-source distributed storage system, providing object (RADOS Gateway), block (RBD), and file (CephFS) storage atop **RADOS (Reliable Autonomic Distributed Object Store)**, which distributes and replicates data across a cluster with no single point of failure. Ceph runs over 100G+ Ethernet and increasingly exploits NVMe-oF (Ceph NVMe-oF gateway) for performance; it is deployed widely (Red Hat Ceph, OpenStack Cinder/Swift backends).
- **Hyperscaler proprietary equivalents**: AWS **EBS** (Elastic Block Store) and **S3** (Simple Storage Service), Azure Blob/Disk, and Google's Colossus and Persistent Disk are the proprietary, massive-scale distributed storage systems that underpin the public clouds — all fundamentally network storage, with the fabric's bandwidth and latency directly determining their performance.

For AI specifically, high-throughput parallel file systems and object stores (Lustre, IBM Storage Scale/GPFS, WEKA, VAST, and cloud object stores) feed training data to GPU clusters, and the storage fabric must sustain the enormous read bandwidth that thousands of GPUs demand — making storage networking an integral part of AI-cluster design, not an afterthought.

## Composable Disaggregated Infrastructure

The frontier is **Composable Disaggregated Infrastructure (CDI)**: decoupling compute, storage, and (via CXL) memory into independent pools connected by fast fabrics, composed on demand into the right machine for each workload. Storage disaggregation is the most mature dimension:
- **JBOF (Just a Bunch of Flash)**: dense flash enclosures accessed over NVMe-oF, separating flash capacity from compute, so capacity and compute scale independently.
- **SmartNIC/DPU NVMe-oF termination**: programmable NICs (NVIDIA BlueField, AMD Pensando, Broadcom's storage SmartNICs) terminate NVMe-oF and present local-looking NVMe to the host, offloading the storage networking from the host CPU and providing an isolation/virtualization boundary.
- **SNIA Swordfish** provides a standardized management API for composable storage, and **CXL** (File 04) extends disaggregation to memory, with CXL-attached storage-class memory and persistent memory blurring the storage/memory boundary.

The disaggregation trend — compute separate from storage capacity separate from memory, all connected by fast RDMA/NVMe-oF/CXL fabrics — is reshaping datacenter architecture, improving utilization (no stranded capacity) and flexibility (compose the right machine per workload), with the **network as the enabling substrate**: disaggregation is only viable because RDMA and NVMe-oF make remote resources nearly as fast as local ones.

## Conclusion

Storage networking has been transformed from a specialized, Fibre-Channel-dominated SAN discipline into a central pillar of datacenter architecture, driven by flash (which moved the bottleneck to the network) and RDMA (which made remote storage nearly as fast as local). The protocol landscape spans the enterprise FC incumbent, the simple-and-ubiquitous iSCSI, and the modern NVMe-over-Fabrics family — NVMe/RDMA for the lowest latency, NVMe/TCP for fabric simplicity, NVMe/FC for FC modernization — atop which distributed systems (Ceph, the hyperscaler stores) and parallel file systems feed data to compute at scale. For AI, the storage fabric must sustain enormous read bandwidth for training data and absorb large checkpoints, making it integral to cluster design. And the disaggregation frontier — JBOF, DPU-terminated NVMe-oF, CXL memory pooling, composable infrastructure — is reshaping the datacenter into pools of compute, storage, and memory composed on demand, viable only because the network has become fast and lossless enough to make remote resources behave like local ones. The next chapter turns to securing all of this: the network security, micro-segmentation, and zero-trust architectures that protect the datacenter fabric.
