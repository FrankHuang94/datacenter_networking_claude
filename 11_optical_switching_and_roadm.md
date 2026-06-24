# Optical Switching, ROADM Architecture, and Reconfigurable Networks

## Introduction: Switching Light Without Touching the Bits

Once data is on a wavelength of light traveling through fiber, it is enormously advantageous to keep it optical for as long as possible. Every conversion from optical to electrical and back (an "OEO" conversion) consumes power, adds latency, and requires expensive transponders. **Optical switching** — routing wavelengths or whole fibers without converting them to electrical signals — lets the network steer light through the topology while it remains light. This chapter covers the technologies of optical switching: the **ROADM (Reconfigurable Optical Add-Drop Multiplexer)** that routes individual wavelengths through the transport network, the **WSS (Wavelength Selective Switch)** at its heart, the **OXC (Optical Cross-Connect)** for large-scale all-optical switching, the **OCS (Optical Circuit Switch)** that hyperscalers now deploy inside datacenters, and the emerging silicon-photonic switches that promise nanosecond reconfiguration. It builds on the WDM and component physics of File 09 and connects to the datacenter topology innovations of File 06 and the AI-fabric reconfigurability of File 15.

## ROADM — Reconfigurable Optical Add-Drop Multiplexer

The **ROADM** is the workhorse of the modern optical transport network. Its job is to take the dozens of wavelengths arriving on incoming fibers and, for each wavelength, decide whether to **pass it through** (continue toward another destination), **drop it** (deliver it to a local transponder/receiver), or **add** a locally generated wavelength to an outgoing fiber — all in the optical domain, without converting the pass-through wavelengths to electrical. This wavelength-granular, remotely reconfigurable routing is what makes a modern optical network agile: operators can provision and reroute wavelengths from a management console, without sending technicians to patch fibers.

### Evolution and Architecture

ROADMs evolved through generations of increasing flexibility. Early fixed OADMs could add/drop only predetermined wavelengths. Modern ROADMs are built around the **Wavelength Selective Switch (WSS)** and add **degrees** — a "degree" is a direction (an incoming/outgoing fiber pair toward a neighboring node). A 2-degree ROADM sits on a linear span; an 8- or 20-degree ROADM sits at a mesh junction routing wavelengths among many directions.

The most flexible modern architecture is the **CDC ROADM (Colorless, Directionless, Contentionless)**:
- **Colorless**: any add/drop port can handle any wavelength (not hardwired to a specific color), so a transponder can be tuned to any wavelength and connected to any port.
- **Directionless**: an added/dropped wavelength can be routed to or from any degree (direction), not just one.
- **Contentionless**: multiple instances of the same wavelength (from different directions) can be added/dropped simultaneously without internal blocking.

A CDC ROADM thus allows fully automated, software-defined wavelength provisioning — any wavelength, any direction, any port — the foundation of software-defined optical networking. ROADMs also incorporate **amplifier stages** (pre-amplifiers, boosters, and mid-stage EDFAs) and must manage the **OSNR budget** as wavelengths cascade through multiple ROADMs, since each ROADM and amplifier adds loss and noise. **Gridless (flex-grid) ROADMs** support variable channel widths (File 09), allocating spectrum flexibly to channels of different baud rates.

## WSS — Wavelength Selective Switch

The **WSS** is the optical component at the heart of every ROADM. It takes a fiber carrying many wavelengths, spatially disperses them (with a grating), and uses a programmable spatial light modulator to independently route each wavelength to any of several output ports. The dominant technology is **LCoS (Liquid Crystal on Silicon)** — a 2D array of liquid-crystal pixels that steer each dispersed wavelength by imposing a programmable phase pattern — which supports the full C-band, 96+ channels, flex-grid operation, and per-wavelength attenuation control (3–5 dB insertion loss per port). An alternative technology is **MEMS-based WSS** (tiny tilting mirrors). 

The WSS market is a near-duopoly: **Lumentum** holds the largest share (~45–50%), followed by **Coherent (the merged II-VI/Finisar)** (~35%), with smaller players (Santec, others). Because every ROADM port needs a WSS, this duopoly gives Lumentum and Coherent enormous leverage in the optical-systems supply chain (File 23). The WSS is one of the highest-value optical components, and the LCoS technology that powers it is a sophisticated photonic-MEMS-LC hybrid.

## OXC — Optical Cross-Connect

An **OXC (Optical Cross-Connect)** performs all-optical switching at the granularity of whole fibers or fiber-spatial-paths (rather than individual wavelengths). The dominant technology for large OXCs is **3D MEMS**: an array of microscopic mirrors that can tilt in two axes to steer a beam of light from any input fiber to any output fiber, building switch matrices as large as **1000×1000 ports** or more. OXCs are used in submarine cable landing stations (to interconnect cable systems and terrestrial backhaul) and in large optical cores. **Calient Technologies** is a leading vendor of 3D-MEMS OXCs (its S-series). OXCs are slow to reconfigure (milliseconds, limited by mirror movement) but offer massive, protocol-transparent, bit-rate-transparent all-optical switching at very low power per bit — switching light without ever looking at it.

## OCS — Optical Circuit Switching in the Datacenter

The most striking recent application of optical switching is **inside the datacenter**, where **Google** pioneered **Optical Circuit Switching (OCS)** at production scale. Google's **Apollo/Palomar** OCS systems, described in the SIGCOMM 2022 paper **"Jupiter Evolving,"** use **MEMS-based optical switches** to dynamically reconfigure the topology of the datacenter fabric. Rather than a fixed Clos with electronic spine switches, Google interposes OCS between aggregation blocks, so the logical topology can be reconfigured (in roughly milliseconds) to match traffic patterns and to allow incremental, heterogeneous upgrades of the fabric.

The motivations are several. **Topology engineering**: different traffic patterns (and, for AI, different collective-communication algorithms) are best served by different topologies, and OCS lets the fabric adapt. **Incremental upgrade**: OCS decouples the generations of equipment, letting Google upgrade parts of the fabric without forklift replacement. **Elephant flows**: large, long-lived flows can be given dedicated optical circuits, offloading them from the packet-switched fabric. **Power and cost**: eliminating a tier of electronic spine switches (replacing it with passive-ish optical switching) saves power and cost. For AI training specifically, the ability to reconfigure topology to match the communication pattern of a given job — providing non-blocking connectivity for the AllReduce or AllToAll a job needs — is a powerful optimization (File 15). Google's deployment of OCS at datacenter scale is one of the most important networking innovations of the past decade, and other operators are following.

## Silicon-Photonic Switches

A frontier technology is the **silicon-photonic switch** — optical switching integrated on a silicon-photonic chip using **microring resonators** or **Mach-Zehnder interferometers (MZIs)** as the switching elements. Unlike MEMS (millisecond reconfiguration) or LCoS (similar), silicon-photonic switches can reconfigure in **nanoseconds**, opening the possibility of fast, fine-grained optical switching that could one day switch individual packets or bursts optically. Current devices are limited in port count (on the order of 64×64) and face challenges (insertion loss, thermal tuning of rings, integration with the rest of the system). A startup ecosystem — **Lightmatter (Passage), Ayar Labs, Celestial AI, Salience Labs** — is developing silicon-photonic switching and interconnect for AI fabrics, where the combination of fast switching and optical bandwidth density could be transformative (Files 13, 24). Silicon-photonic switching is also intimately tied to co-packaged optics, where the optical engine and switching could one day be integrated with the compute.

## Optical Switching Vendors and Systems

The optical-systems vendors that build ROADMs, OXCs, and the surrounding transport platforms include:
- **Ciena** — Waveserver and GeoMesh platforms, a leader in coherent transport and ROADM systems.
- **Nokia** — the 1830 Photonic Service Switch (PSS) family and 1350 management.
- **Infinera** — the GX series and FlexILS open line system, with its distinctive InP PIC technology.
- **Fujitsu** — the 1FINITY platform, prominent in disaggregated/open optical.
- **Huawei** — OptiX OSN 9800 and related platforms, dominant in many non-Western markets but constrained by export controls in others.
- **ZTE**, **Coriant (now part of Infinera)**, and others round out the field.

Component suppliers — **Lumentum and Coherent** (WSS, amplifiers, lasers), **Calient** (OXC) — supply the building blocks that these systems vendors integrate. The competitive dynamics, including the tension between integrated systems and disaggregated open-optical architectures, are detailed in File 23.

## Conclusion

Optical switching is the art of routing light without touching the bits — steering wavelengths and fibers through the network while they remain in the optical domain, saving the power, latency, and cost of electrical conversion. The ROADM, built on the WSS, makes the transport network agile and software-defined; the OXC provides massive all-optical switching at cable scale; and, most strikingly, optical circuit switching has migrated inside the datacenter, where Google's OCS deployment reconfigures the fabric topology dynamically to serve changing traffic and AI collective patterns. The frontier — nanosecond silicon-photonic switching — promises to push optical switching toward ever-finer granularity, blurring the line between circuit and packet switching and intertwining with the co-packaged-optics revolution. As AI fabrics demand both enormous bandwidth and topological flexibility, optical switching is moving from the periphery of the transport network to the heart of the datacenter, and the technologies surveyed here are increasingly central to how the largest AI clusters are built. The next chapter follows the wavelengths these switches route out of the datacenter entirely — into the metro, long-haul, and submarine systems that constitute datacenter interconnect.
