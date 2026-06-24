# Coherent Optical, Transceivers, and the Pluggable Ecosystem

## Introduction: From Power to Field

The single most important technological transition in optical networking over the past fifteen years was the move from **direct detection** — sensing only the optical power — to **coherent detection** — recovering the full complex optical field, amplitude and phase, on both polarizations. This transition, enabled by high-speed analog-to-digital converters and powerful digital signal processing, transformed optical transmission. It allowed dispersion and PMD to be compensated electronically rather than optically, enabled high-order modulation formats that pack many bits per symbol, doubled capacity through polarization multiplexing, and pushed reach from hundreds to thousands of kilometers. And in the 2020s, coherent technology underwent a second transformation: it shrank from rack-sized line cards into **pluggable modules** (400G-ZR and beyond) the size of a thick USB stick, collapsing the boundary between the optical-transport world and the datacenter-switching world.

This chapter develops coherent optical technology in depth — the DSP, the modulation formats, the FEC, the constellation shaping — then surveys the transceiver form factors and MSA standards, the ZR/ZR+ datacenter-interconnect ecosystem, the coherent DSP ASIC vendors, and the open-optical-networking movement that is disaggregating the optical network. It builds directly on the optical physics of File 09 and feeds the DCI and submarine coverage of File 12 and the transceiver-market analysis of File 21.

## Coherent Optical Technology Deep Dive

### The DSP at the Heart of Coherent

A coherent transceiver is, fundamentally, a high-speed mixed-signal system wrapped around a powerful **DSP (Digital Signal Processor)**. On the receive side, the coherent front end (90° hybrid, balanced photodetectors, File 09) delivers four electrical signals (I and Q for each polarization) to **ADCs sampling at 100+ gigasamples per second**, and the DSP then performs an extraordinary sequence of operations to recover the data:

- **Chromatic dispersion compensation**: a frequency-domain equalizer applies a digital filter that exactly inverts the fiber's accumulated dispersion — undoing thousands of km of pulse broadening in the digital domain. Because dispersion is linear and stable, this compensation is near-perfect.
- **Polarization demultiplexing and PMD compensation**: an adaptive "butterfly" equalizer (a 2×2 MIMO filter), typically driven by the Constant Modulus Algorithm (CMA), separates the two polarization-multiplexed signals and tracks the time-varying polarization rotation and PMD.
- **Carrier frequency and phase recovery**: algorithms (Viterbi-Viterbi, blind phase search) estimate and remove the frequency offset between the transmit laser and the local oscillator, and track the phase noise of both lasers.
- **Nonlinear compensation (optional, costly)**: digital back-propagation (DBP) or Volterra nonlinear equalizers can partially undo fiber nonlinearity by simulating the inverse propagation, at high computational cost.
- **Soft-decision FEC decoding**: the final stage applies powerful FEC (below).

This DSP is implemented in a leading-edge ASIC (often the most advanced process node available), and it is the principal determinant of a coherent transceiver's reach, capacity, and power. The progression of coherent DSPs — from 100G to 400G to 800G to 1.6T — is a story of ever-faster data converters, ever-more-powerful equalization and FEC, and ever-finer process nodes.

### Modulation Formats and the Reach-Capacity Trade-off

Coherent transmission's flexibility comes from its ability to choose a **modulation format** that trades spectral efficiency (bits per symbol) against required signal-to-noise ratio (and thus reach). All formats are dual-polarization (PM, polarization-multiplexed), doubling the bits per symbol:

| Format | Bits/symbol (×2 pol) | Typical reach / capacity |
|---|---|---|
| PM-BPSK | 2 | Longest reach; submarine 100G |
| PM-QPSK | 4 | 100G/wavelength; transcontinental DWDM |
| PM-8QAM | 6 | 150–200G at medium reach |
| PM-16QAM | 8 | 400G at ~1000 km (with SD-FEC, Raman) |
| PM-64QAM | 12 | 600G at 300–500 km; 800G at ~80 km |
| PM-128QAM | 14 | 1.2T+ experimental, research stage |

The fundamental trade-off: higher-order formats (more bits per symbol, more capacity) pack the constellation points closer together, requiring higher SNR and thus shorter reach. A submarine cable spanning an ocean might use PM-QPSK (robust, long reach); a metro DCI link of 80 km might use PM-64QAM (high capacity, short reach). Coherent transceivers can often switch formats in software, adapting to the link.

### Probabilistic Constellation Shaping

A powerful refinement is **Probabilistic Constellation Shaping (PCS)**: rather than transmitting all constellation points with equal probability, PCS transmits the inner (lower-energy) points more frequently than the outer points, approximating the Gaussian distribution that information theory says is optimal. This **shaping gain** yields roughly 1–1.5 dB of improvement over uniform QAM — equivalent to meaningfully more reach or capacity — and, crucially, allows **fine-grained, near-continuous adjustment** of the effective bit rate (by tuning the shaping distribution) rather than the coarse steps of discrete modulation formats. PCS is implemented in the coherent DSPs of all the leading vendors (Acacia/Cisco, Ciena, Nokia, Infinera) and is one of the key technologies that let modern coherent systems operate close to the Shannon limit and adapt precisely to each link's conditions.

### Baud Rate Scaling

In parallel with higher-order modulation, coherent systems scale the **baud rate** (symbols per second): 32 Gbaud (100G era) → 64 Gbaud (400G) → ~96 Gbaud (800G) → ~130 Gbaud (1.2T–1.6T). Higher baud rates demand higher-bandwidth, higher-sample-rate data converters but allow more data per wavelength without resorting to the highest-order (shortest-reach) modulation. A 130 Gbaud × PM-64QAM signal carries about 1.56 Tbps gross. The march to higher baud rates is bounded by the bandwidth and effective-number-of-bits of the ADCs/DACs and the bandwidth of the modulators (driving the adoption of thin-film lithium niobate and advanced InP modulators, File 09).

### FEC in Coherent Systems

Coherent systems use the most powerful FEC in networking, because they operate at very low OSNR. The progression:
- **Hard-decision FEC (~7% overhead, ITU-T G.975.1)**: corrects to a pre-FEC BER around 3×10⁻³.
- **Soft-decision FEC (SD-FEC, ~15–20% overhead)**: typically LDPC (Low-Density Parity-Check) or turbo codes, exploiting soft (probabilistic) information from the demodulator; tolerates a pre-FEC BER around 2×10⁻², delivering a **Net Coding Gain (NCG)** of about 11–12 dB.
- **Higher-overhead codes (~27%)**: turbo product codes and advanced LDPC, used in the highest-performance systems (e.g., Ciena's WaveLogic).

That 11+ dB of net coding gain is enormous — it opens up an equivalent amount of optical margin, directly translating into longer reach or higher capacity. The combination of SD-FEC and PCS is what lets modern 800G coherent systems achieve reaches that would have been unthinkable a decade ago.

## Transceiver Form Factors and MSA Standards

The physical packaging of optics into hot-pluggable modules is governed by **Multi-Source Agreements (MSAs)** — industry agreements that standardize a form factor so multiple vendors can build interchangeable modules. The form-factor progression tracks the speed progression:

- **QSFP28 (100G)** — 4×25G NRZ; the dominant 100G form factor since ~2016; ~3.5 W; hosts 100G-SR4, LR4, CWDM4.
- **QSFP56 (200G)** — 4×50G PAM4; same mechanical envelope as QSFP28; saw limited adoption (the industry largely skipped 200G).
- **QSFP-DD (400G)** — "Double Density," 8 electrical lanes (8×50G PAM4 for 400G); backward compatible with QSFP28/QSFP56 (a QSFP28 module works in a QSFP-DD cage); up to ~14 W; the dominant 400G form factor, hosting 400G-DR4, FR4, LR4, and even 400G-ZR/ZR+.
- **OSFP (400G/800G)** — "Octal Small Form-factor Pluggable," physically larger than QSFP-DD with a higher power budget (up to ~24 W, important for coherent ZR+); not mechanically interchangeable with QSFP-DD; favored where power headroom matters and for 800G.
- **QSFP-DD800 (800G)** — QSFP-DD mechanicals with 8×100G PAM4; ~18–20 W; adopted by Cisco, Arista, Juniper for 800G in the QSFP-DD ecosystem.
- **OSFP800 (800G)** — OSFP form factor at 800G, with more thermal headroom for coherent 800G-ZR+.
- **Legacy CFP family (CFP, CFP2, CFP4, CFP8)** — older, larger coherent form factors; **CFP2-ACO** (Analog Coherent Optics, with the DSP on the host) and **CFP2-DCO** (Digital Coherent Optics, DSP in the module) carried early 100G/200G coherent; being displaced by QSFP-DD and OSFP coherent pluggables.
- **DSFP, SFP-DD** — smaller form factors for lower-speed (e.g., 2×25G/50G) niche applications.

The trend is unmistakable: each generation packs more electrical lanes (and higher per-lane rates) into a hot-pluggable module, with power budget the binding constraint — which is precisely why linear-drive optics (LPO) and co-packaged optics (CPO) emerge to break the power wall at 800G and 1.6T (File 13).

## ZR and ZR+ — The DCI Revolution

The most disruptive development in coherent optics was packing it into a pluggable and standardizing it for **datacenter interconnect (DCI)**. Before ZR, connecting two datacenters over dark fiber required expensive, rack-scale DWDM transport systems. ZR put coherent transmission in a QSFP-DD/OSFP module that plugs directly into a router or switch.

### 400G-ZR

**400G-ZR**, standardized by the **OIF (Optical Internetworking Forum)**, carries 400G on a single coherent wavelength using **PM-16QAM at ~60 Gbaud**, reaching **80 km over a single unamplified DWDM span** (and 300–600 km with EDFA amplification in a metro network). It fits in a QSFP-DD or OSFP pluggable drawing 14–17 W, integrating a DP-IQ modulator, a coherent receiver, and a coherent DSP. Critically, 400G-ZR is an **interoperability standard** — modules from different vendors can interoperate — which commoditized coherent DCI and let hyperscalers buy coherent optics like any other pluggable. This was a profound shift in the optical-transport business model (below).

### ZR+ and OpenZR+

**ZR+** (formalized by the **OpenZR+ MSA**) extends 400G-ZR with higher-performance DSP for longer reach and more flexibility: it supports lower-order formats (PM-QPSK for very long reach), higher-order formats (PM-8QAM, PM-16QAM), SD-FEC with PCS, and amplified reaches up to ~3000 km. OpenZR+ members include Acacia (Cisco), Coherent (formerly II-VI), InnoLight, and others. The higher power budget of OpenZR+ favors the OSFP form factor. A 400G-ZR+ link can span hundreds of km through a ROADM network, bringing coherent flexibility to the pluggable form factor.

### 800G-ZR+

**800G-ZR+**, emerging in 2024–2025, carries 800G on a single wavelength using **96–130 Gbaud × PM-64QAM**, reaching ~80 km unamplified and 500–800 km amplified, in OSFP/OSFP800 with DSP power consumption around 15–20 W. Vendors include Acacia (Cisco's "Orion" generation), Coherent, and Nokia/Acacia. 800G-ZR+ extends the pluggable-coherent revolution to the next speed, intensifying the competition between pluggable coherent and traditional integrated transport systems (File 23).

## Coherent DSP ASIC Vendors

The coherent DSP is the crown jewel of the coherent ecosystem — the component that most determines performance — and a handful of vendors compete fiercely:

- **Cisco/Acacia**: Cisco acquired **Acacia in 2021 for ~$4.5 billion**, vertically integrating a leading coherent-DSP and pluggable maker. Acacia's DSPs (the "Tierra," "Orion," and successor generations spanning 400G, 800G, and 1.2T) power Cisco's coherent line cards and pluggables, and Acacia continues to sell modules to third parties — making Cisco both a systems vendor and a merchant coherent supplier.
- **Ciena (WaveLogic)**: Ciena's **WaveLogic** DSPs are widely regarded as best-in-class for reach × capacity. **WaveLogic 5 Extreme (WL5e)** achieves 400G at ~6000 km, 800G at ~3000 km, and 1.2T at shorter reach, using advanced PCS and SD-FEC; **WaveLogic 6** targets 1.6T per wavelength. WaveLogic is proprietary to Ciena's systems and a core competitive moat (File 23).
- **Nokia (Photonic Service Engine, PSE)**: Nokia's Bell Labs-derived **PSE** DSPs (PSE-3, PSE-4, and successors) power its 1830 optical platforms, with virtual variants (PSE-V) for white-box transponders.
- **Infinera (ICE)**: Infinera's **Infinite Capacity Engine** DSPs (ICE6 at ~800G/wavelength, ICE7 toward 1.6T) are paired with Infinera's own **InP photonic integrated circuits (PICs)** — a vertically integrated photonics-plus-DSP approach distinctive in the industry (File 23).
- **Marvell**: Marvell, through its **Inphi acquisition ($10 billion, 2021)**, offers coherent DSPs (the "Meru" 400G and "Viper" 800G families) sold to module makers (InnoLight, Coherent, and others), making it a major merchant coherent-DSP supplier for the pluggable ecosystem.
- **Broadcom**: Broadcom competes in coherent DSP and high-volume pluggable optics, leveraging its broader optical-component portfolio.
- **HiSilicon (Huawei)**: Huawei's internal coherent DSPs power its OTN and metro systems; advanced but unavailable to third parties and constrained by US export controls.

The distinction between **merchant DSP suppliers** (Marvell/Inphi, Acacia, Broadcom — selling to module makers) and **integrated systems vendors** (Ciena, Nokia, Infinera — keeping their DSPs in-house) is a central axis of the coherent market, and the rise of merchant DSPs in standardized pluggables (ZR/ZR+) is what enabled the open-optical disaggregation discussed next.

## Open Optical Networking and White-Box Transponders

The traditional optical-transport business was **vertically integrated**: a vendor sold the transponders, the ROADMs, the amplifiers, and the management software as a single proprietary system. The pluggable-coherent revolution and the hyperscalers' preference for open, multi-sourced infrastructure are dismantling this model.

- **OpenROADM** is an MSA defining open **YANG models** for ROADM and transponder control, enabling multi-vendor optical networks; adopted by carriers such as AT&T and Deutsche Telekom.
- **The Telecom Infra Project (TIP)**, led by Meta, drives **Open Optical & Packet Transport (OOPT)** — disaggregated optical transponder and line-system specifications and white-box hardware running third-party coherent pluggables.
- **Open Line Systems (OLS)** decouple the optical line system (ROADMs, EDFAs, the optical supervisory channel) from the transponders. In a disaggregated architecture, an operator buys the OLS from one vendor and plugs coherent ZR/ZR+ wavelengths (from routers/switches, or from third-party transponders) into it, with the OLS managing wavelength routing and amplification. OpenROADM-compatible OLS products come from Fujitsu, Ciena, Lumentum, and others.

The **business-model impact** is significant. Pluggable coherent (ZR/ZR+) lets operators put the coherent function directly in their routers/switches, bypassing standalone transponder systems for many DCI applications — a direct threat to the integrated-transport revenue of Ciena, Nokia, and Infinera, who are adapting by offering their own pluggable-compatible platforms and emphasizing the longer-reach, higher-capacity applications where integrated systems with the best DSPs (WaveLogic, ICE) still win. Component suppliers (Lumentum, Coherent) benefit from selling into both the pluggable and the integrated worlds. And the hyperscalers — Google, Meta, Microsoft — increasingly build their own open line systems and buy coherent pluggables in volume, driving the disaggregation that reshapes the entire optical-transport industry (File 23).

## Extended Deep Dive: The Coherent Receiver DSP Pipeline in Detail

It is worth tracing the coherent receive DSP pipeline in more detail, because it is where the magic of coherent transmission actually happens and because its computational cost drives the power and process-node requirements of every coherent transceiver. After the coherent front end (the 90° hybrid and four balanced photodetectors, File 09) delivers the four electrical tributaries (in-phase and quadrature for each of the two polarizations) to the ADCs, the digital pipeline proceeds roughly as follows.

First, **front-end correction** compensates for imperfections in the analog hardware itself: skew between the four tributaries, gain imbalance, quadrature imbalance (the I and Q not being exactly 90° apart), and ADC nonlinearity. These corrections (often via a Gram-Schmidt orthogonalization or a dedicated calibration) are essential because the downstream algorithms assume an ideal received field.

Second, **static chromatic dispersion compensation** applies a fixed (per-link) frequency-domain filter. Because dispersion is a linear, all-pass phase distortion with a known quadratic phase-versus-frequency characteristic, it can be inverted exactly by a filter implemented efficiently via overlap-and-save FFT processing. The length of this filter grows with the accumulated dispersion (and thus the link distance and baud rate), so longer links and higher baud rates demand more FFT taps — a major contributor to DSP gate count and power. A trans-oceanic link may require a dispersion-compensation filter spanning thousands of taps.

Third, the **adaptive equalizer** — a 2×2 (or 4×4 for advanced schemes) MIMO butterfly filter — simultaneously performs polarization demultiplexing (separating the two polarization-multiplexed signals that the fiber has rotated and mixed) and dynamic equalization (tracking time-varying PMD and residual dispersion). It is typically initialized by the **Constant Modulus Algorithm (CMA)** (which exploits the constant-modulus property of PSK signals) and then switches to a **decision-directed LMS (Least Mean Squares)** mode for the data-carrying QAM signal. This equalizer adapts continuously, tracking the fiber's slowly varying polarization state.

Fourth, **carrier recovery** removes the frequency offset between the transmit laser and the local oscillator (which can be hundreds of MHz to GHz) and tracks the phase noise of both lasers. Frequency-offset estimation is often done in the frequency domain (finding the spectral peak of the signal raised to the fourth power for QPSK), and phase tracking uses a **Viterbi-Viterbi** or **blind phase search (BPS)** algorithm; for higher-order QAM, the phase-noise tolerance shrinks, demanding narrow-linewidth lasers and more sophisticated phase estimation.

Fifth and finally, **soft-decision FEC decoding** (LDPC or turbo product codes) uses the soft (log-likelihood-ratio) information from the demodulator to correct errors at pre-FEC BERs as high as ~2×10⁻², delivering the 11+ dB net coding gain that opens the optical margin (above). Optional **nonlinear compensation** (digital back-propagation) can precede the FEC at high computational cost.

The aggregate computational load of this pipeline — running at 100+ gigasamples per second across four tributaries — is staggering, which is why coherent DSPs are fabricated on the most advanced available process nodes and why their power consumption (15–25 W for an 800G coherent DSP) is the dominant term in a coherent pluggable's power budget and the reason coherent pluggables need the higher power envelopes of OSFP.

## Extended Deep Dive: Optical Signal-to-Noise Ratio and the Link Budget

The currency of optical transmission is **OSNR (Optical Signal-to-Noise Ratio)**, and every coherent link is, at bottom, an OSNR budget. Each EDFA in a chain adds ASE noise (File 09), and the OSNR degrades with each span; the question is whether the accumulated OSNR at the receiver, after the DSP's coding gain, suffices to recover the chosen modulation format at the target post-FEC BER. Higher-order modulation (more bits per symbol) requires higher OSNR — roughly 3 dB more per doubling of constellation density — which is why PM-64QAM reaches only ~80 km while PM-QPSK crosses oceans. The link designer trades modulation order (capacity) against the number and spacing of amplified spans (reach), with PCS providing fine-grained adjustment between the discrete modulation steps and SD-FEC providing the coding-gain margin. Raman amplification (File 09) improves the effective OSNR on long spans by amplifying the signal before it has fully attenuated, extending reach where EDFA-only budgets fall short. Understanding the link as an OSNR budget — generation minus accumulated noise minus penalties, versus the required OSNR for the format plus FEC — is the unifying framework for all coherent system design, from a metro DCI hop to a trans-Pacific submarine cable.

## Extended Deep Dive: The Pluggable-Coherent Disruption in Numbers

The disruptive economics of pluggable coherent deserve quantification. A traditional metro DWDM deployment required a chassis-based transport system: line cards with embedded coherent transponders, a separate ROADM, amplifiers, and a management plane — capital equipment costing tens to hundreds of thousands of dollars per node, plus the operational overhead of a separate transport network and the specialized staff to run it. The 400G-ZR pluggable collapses much of this: a coherent wavelength now fits in a QSFP-DD/OSFP module costing a small fraction of a transponder line card, plugged directly into the router or switch the operator already runs, managed through the same tooling. For a hyperscaler connecting datacenters across a metro at 400G, the ability to add a coherent wavelength by inserting a pluggable into a router port — rather than provisioning a separate transport system — is a step-change in cost, simplicity, and time-to-deploy. This is why the pluggable-coherent transition reshaped the optical-transport business model (above): it moved a high-value function (coherent transmission) from a specialized, integrated, high-margin transport system into a commoditized, standardized, multi-sourced pluggable, with the value migrating from the systems vendors toward the DSP and module makers and toward the router/switch vendors whose ports now host the coherent function. The integrated-systems vendors (Ciena, Nokia, Infinera) retain the longest-reach, highest-capacity applications — where their best-in-class DSPs (WaveLogic, PSE, ICE) and integrated photonics still win — but the metro and regional tiers are increasingly the domain of the pluggable, and the strategic battle between integration and disaggregation (File 23) is, in large part, this story.

## Conclusion

Coherent optical technology, born in the long-haul transport network, has become one of the defining forces in datacenter networking — first by enabling the capacity and reach that connect datacenters across metros, regions, and oceans, and then, through the pluggable ZR/ZR+ revolution, by collapsing the boundary between the optical-transport world and the datacenter-switching world. The coherent DSP — with its dispersion compensation, polarization tracking, high-order modulation, probabilistic constellation shaping, and powerful soft-decision FEC — is the engine of this transformation, and the competition among its makers (Ciena, Acacia/Cisco, Nokia, Infinera, Marvell, Broadcom) is among the most technically demanding in the industry. The packaging of coherent into pluggable modules, and the open-optical movement disaggregating the transport network, are reshaping a decades-old business model. And the relentless scaling — 400G to 800G to 1.6T per wavelength, ever-higher baud rates and modulation orders — runs into the same power wall that drives the rest of optics toward linear-drive and co-packaged integration. The chapters ahead build on this: File 11 on the optical switching (ROADMs, OXC, OCS) that routes these coherent wavelengths; File 12 on the DCI and submarine systems they traverse; and File 13 on the co-packaged future that will pull coherent and direct-detect optics alike inside the switch package.
