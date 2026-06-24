# Optical Networking Fundamentals — Physics, Components, and Technologies

## Introduction: Why Light

Beyond the reach of copper — past a few meters at modern signaling rates — the datacenter network becomes optical. Light carried in glass fiber is the only medium that can move data the distances a datacenter, a campus, a metro region, or an ocean demands, at the bandwidths the modern world consumes, with acceptable energy and loss. Optical networking is where networking meets fundamental physics: the behavior of electromagnetic waves in dielectric waveguides, the quantum mechanics of light amplification, the nonlinear response of glass to intense optical fields, and the information-theoretic limits on how many bits a fiber can carry. This chapter builds optical networking from its physical foundations — the properties of light and fiber, the impairments that limit transmission, the components that generate, modulate, amplify, and detect light, and the wavelength-division-multiplexing techniques that pack many channels into a single fiber. It is prerequisite to the coherent-optics, optical-switching, DCI, and co-packaged-optics chapters that follow (Files 10–13).

## Light as a Communication Medium — The Physics

### The Electromagnetic Spectrum and the Telecom Windows

Optical communication uses near-infrared light, in the wavelength range of roughly 1260–1625 nm — invisible to the human eye (visible light spans ~400–700 nm) but ideally suited to silica glass fiber. Within this near-infrared range, the industry has defined named **bands**:
- **O-band (1260–1360 nm)** — "Original," near the zero-dispersion wavelength of standard fiber; used for short-reach datacenter optics (e.g., 1310 nm in DR/FR/LR transceivers) because dispersion is minimal there.
- **E-band (1360–1460 nm)** — "Extended," historically degraded by the water-absorption peak (below).
- **S-band (1460–1530 nm)** — "Short," a frontier for future capacity expansion.
- **C-band (1530–1565 nm)** — "Conventional," the dominant band for long-haul and high-capacity transmission.
- **L-band (1565–1625 nm)** — "Long," used to expand capacity beyond the C-band.

The **C-band's dominance** for long-haul and DWDM is not arbitrary. Two facts make it special: silica fiber has its **minimum attenuation** — about 0.2 dB/km — around 1550 nm, in the C-band, so signals travel farthest there; and the **erbium-doped fiber amplifier (EDFA)**, the workhorse optical amplifier, happens to provide gain precisely in the C-band (1530–1565 nm). The coincidence of minimum loss and a practical amplifier in the same band is the physical foundation of the entire long-haul optical industry. The L-band extends this with a second amplification window, and the S-band is the next frontier.

### Fiber Attenuation

A signal in fiber weakens as it propagates, and three mechanisms dominate this **attenuation**:
- **Rayleigh scattering** — the scattering of light off microscopic density fluctuations frozen into the glass, scaling as λ⁻⁴ (so it dominates at shorter wavelengths and falls rapidly toward longer ones). Rayleigh scattering sets the fundamental loss floor at the wavelengths below ~1500 nm.
- **Infrared absorption** — the glass itself absorbs light at long wavelengths (above ~1700 nm), as the photon energy couples to molecular vibrations. This sets the loss floor at long wavelengths.
- **OH⁻ (water) absorption** — hydroxyl ions, contaminants from water in the manufacturing process, absorb strongly with a peak around **1383 nm**, historically creating a "water peak" that made the E-band unusable. Modern **low-water-peak fiber (ITU-T G.652.D)** and bend-insensitive fiber (G.657) eliminate this peak, opening the full spectrum.

The interplay of these mechanisms produces the famous attenuation curve with its minimum near 1550 nm. **Ultra-low-loss (ULL) fiber** — Corning's SMF-28 ULL, Sumitomo's Z-fiber, and similar — pushes the 1550 nm attenuation down to about **0.148–0.16 dB/km** by using a pure-silica core that minimizes Rayleigh scattering, a crucial advantage for submarine and ultra-long-haul links where every fraction of a dB/km extends the unrepeatered or unamplified span.

### Chromatic Dispersion

**Chromatic dispersion** is the phenomenon that different wavelengths travel at slightly different speeds in fiber. Since any real optical pulse contains a spread of wavelengths (and any modulated signal occupies a band of frequencies), the components arrive at slightly different times, **broadening the pulse** as it propagates and eventually causing adjacent pulses to overlap — inter-symbol interference that limits the achievable baud-rate × distance product. Dispersion is characterized by the parameter **D** (in ps/nm/km): for standard single-mode fiber it is approximately **+17 ps/nm/km at 1550 nm**, and it crosses **zero at about 1310 nm** (the "zero-dispersion wavelength," which is why 1310 nm is used for short-reach datacenter optics — minimal dispersion there allows simple direct-detection transmission).

Dispersion can be managed several ways. Historically, **Dispersion-Compensating Fiber (DCF)** — fiber with large negative dispersion — was spliced into links to cancel accumulated dispersion, and **Fiber Bragg Gratings (FBGs)** provided compact compensation. The modern coherent approach is far more elegant: **electronic dispersion compensation in DSP**. Because a coherent receiver recovers the full optical field (amplitude and phase), it can apply a digital filter that exactly inverts the (linear, well-characterized) dispersion of the fiber — undoing thousands of km of accumulated dispersion in the digital domain, eliminating the need for physical compensation entirely. This is one of the great advantages of coherent detection (File 10).

### Polarization Mode Dispersion

Single-mode fiber actually supports two polarization modes, and imperfections and stresses in real fiber cause these two modes to travel at slightly different speeds — **Polarization Mode Dispersion (PMD)**. PMD is characterized by a coefficient around 0.05–0.2 ps/√km (note the square-root dependence on distance, reflecting its random, statistical nature) and is **temperature- and time-dependent**, varying as the fiber is disturbed. PMD was a serious limitation for high-speed direct-detection systems, but coherent receivers handle it gracefully through **electronic polarization tracking** — the DSP continuously estimates and compensates the polarization state, again exploiting the recovered optical field. This is why coherent systems can use **polarization-division multiplexing** (transmitting independent data on the two polarizations to double capacity) that would be hopeless with direct detection.

### Nonlinear Effects

At the low optical powers of short links, fiber behaves linearly. But long-haul DWDM systems launch substantial power (to overcome loss and maintain signal-to-noise ratio over many amplified spans), and at high power the glass responds **nonlinearly** through the **Kerr effect** (the refractive index depends on optical intensity) and scattering processes:
- **Self-Phase Modulation (SPM)** — a pulse's own intensity modulates the refractive index it experiences, broadening its spectrum.
- **Cross-Phase Modulation (XPM)** — the intensity of one WDM channel modulates the phase of neighboring channels, causing crosstalk.
- **Four-Wave Mixing (FWM)** — three optical waves mix to generate a fourth at a new frequency, which can land on and corrupt another channel. FWM is worst when dispersion is low (which is why zero-dispersion-shifted fiber, G.653, proved problematic for DWDM).
- **Stimulated Raman Scattering (SRS)** — transfers power from shorter-wavelength to longer-wavelength channels (also the basis of Raman amplification, below).
- **Stimulated Brillouin Scattering (SBS)** — limits the power that can be launched into a single narrow-linewidth wavelength (threshold around 10 dBm), as power above the threshold is reflected backward.

These nonlinearities impose the **nonlinear Shannon limit**: increasing launch power improves signal-to-noise ratio (good) but also increases nonlinear noise (bad), so there is an optimal launch power beyond which capacity decreases. This nonlinear limit caps the practical capacity of a fiber at roughly 100 Tbps per fiber pair in the C-band — a ceiling that drives the industry toward C+L+S band expansion, space-division multiplexing, and new fiber types (File 24). Coherent systems can partially compensate nonlinearity in DSP (digital back-propagation), but only at high computational cost.

## Fiber Types

The ITU-T G.65x series standardizes the fiber types that constitute the world's optical plant:
- **G.652 (Standard Single-Mode Fiber, SMF)** — the most-installed fiber worldwide; mode-field diameter ~9.2 µm at 1310 nm; attenuation ~0.18–0.20 dB/km at 1550 nm; zero dispersion at ~1310 nm. The **G.652.D** variant has a suppressed water peak, opening the full spectrum.
- **G.653 (Dispersion-Shifted Fiber, DSF)** — shifts the zero-dispersion wavelength to 1550 nm, which seemed ideal for single-channel 1550 nm systems but proved disastrous for DWDM because zero dispersion maximizes four-wave mixing. Largely obsolete.
- **G.655 (Non-Zero Dispersion-Shifted Fiber, NZ-DSF)** — engineered for a small positive dispersion at 1550 nm (3–8 ps/nm/km), enough to suppress FWM while keeping dispersion low; products like Corning LEAF and Lucent TrueWave. Common in legacy DWDM builds.
- **G.654 (Cut-off-Shifted / Ultra-Low-Loss)** — pure-silica-core fiber with very low attenuation (0.15–0.16 dB/km), used in submarine cables and long-haul terrestrial routes where span loss is the binding constraint.
- **G.657 (Bend-Insensitive)** — engineered for low loss under tight bends, important for the cramped, tightly routed cabling inside datacenter cabinets and access deployments.
- **Multimode fiber (OM1–OM5)** — larger-core fiber (50 µm or 62.5 µm) that supports many propagation modes, used with low-cost VCSELs at 850 nm for short reaches. **OM3** (300 m at 10G, 100 m at 40G-SR4), **OM4** (improved, 100–150 m at higher speeds), and **OM5** (wideband, supporting short-wavelength WDM) serve datacenter intra-rack and TOR-to-server links. Multimode's reach shrinks with speed (modal dispersion limits it), which is why single-mode is taking over even at short reaches in the highest-speed generations.

## Optical Amplifiers

The invention of the optical amplifier — which boosts an optical signal directly, without converting it to electrical and back — is what made long-haul DWDM economically viable, because a single amplifier can boost all WDM channels simultaneously.

### EDFA — The Workhorse

The **Erbium-Doped Fiber Amplifier (EDFA)** is the foundation of long-haul optics. A length of fiber doped with erbium ions is **pumped** by a laser (at 980 nm or 1480 nm), exciting the erbium ions; when signal light in the C-band passes through, it stimulates the excited ions to emit, amplifying the signal directly in the optical domain. EDFAs provide **20–40 dB of gain** with a **noise figure of 4–6 dB**, amplify the entire C-band at once (bandwidth ~35 nm), and are the reason a DWDM system can carry dozens of wavelengths over thousands of km with periodic amplification.

EDFAs have two important imperfections. First, they add **ASE (Amplified Spontaneous Emission)** noise, which accumulates over a chain of amplifiers and ultimately limits the number of spans (and thus the reach) — the optical signal-to-noise ratio (OSNR) degrades with each amplifier. Second, their **gain is not flat** across the C-band, so a chain of EDFAs would progressively tilt the spectrum; this is corrected with **Gain-Flattening Filters (GFF)** and dynamic gain equalization (often integrated into ROADMs). **C+L-band EDFAs** extend amplification into the L-band, roughly doubling the usable spectrum and hence the fiber capacity.

### Raman Amplification

The **Raman amplifier** exploits Stimulated Raman Scattering: a high-power pump laser, at a wavelength ~100 nm shorter than the signal, transfers energy to the signal as both propagate through the transmission fiber itself. Because the gain is **distributed** along the transmission fiber (rather than lumped in a discrete erbium coil), Raman amplification produces a lower effective noise figure — distributed Raman can achieve an effective noise figure *below* the quantum limit of a discrete amplifier, because the signal is amplified before it has fully attenuated. Raman amplification is used on **ultra-long spans (500+ km)** and in **submarine** systems, often in hybrid Raman+EDFA configurations, to extend reach where EDFA-only OSNR would be insufficient.

### SOA and TDFA

The **Semiconductor Optical Amplifier (SOA)** is a chip-scale amplifier (an active laser-like waveguide without mirrors) with fast gain dynamics (nanosecond response). Its speed makes it useful for **optical switching** and burst-mode applications, but its **pattern-dependent gain saturation** (the gain depends on recent signal history) causes distortion in WDM transmission, limiting its use as a transmission amplifier. SOAs are increasingly important in **photonic integration** (they can be integrated on InP photonic chips) and co-packaged optics. The **Thulium-Doped Fiber Amplifier (TDFA)** provides amplification in the **S-band (1450–1530 nm)**, a key enabler for future S+C+L-band capacity expansion (File 24), though it is not yet in widespread commercial deployment.

## Optical Modulators and Transceivers

Generating an optical signal requires a light source (laser) and a way to imprint data onto it (modulation). The choice of modulation technology profoundly affects reach, speed, cost, and power.

### Direct Modulation

The simplest approach is **direct modulation**: vary the laser's drive current to vary its output power. This is cheap and compact, but modulating the current also unintentionally modulates the laser's frequency — **chirp** — which interacts with fiber dispersion to limit reach. Directly modulated lasers (DMLs) are limited to roughly 10G NRZ over single-mode fiber (chirp-limited) but are widely used in **VCSELs** (Vertical-Cavity Surface-Emitting Lasers) at 850 nm for multimode short reaches, where they reach 100G PAM4 over ~100 m of OM4. VCSELs are the low-cost workhorse of intra-datacenter optics, though scaling them to 200G per lane for 1.6T is a serious challenge (File 21).

### Mach-Zehnder Modulators

For higher speeds and longer reaches, **external modulation** separates the (continuously emitting) laser from the modulator. The **Mach-Zehnder Modulator (MZM)** is an optical interferometer: the light is split into two arms, a voltage applied to one arm shifts its phase via the **electro-optic effect**, and when the arms recombine, constructive or destructive interference modulates the output. At the half-wave voltage **Vπ**, the two arms are fully out of phase and the output goes dark. MZMs produce **chirp-free** modulation and can generate complex multilevel formats. 

The classic MZM material is **lithium niobate (LiNbO₃)** — low insertion loss, broadband, but a relatively high Vπ (~4 V) and bulky. The transformative recent development is **thin-film lithium niobate (TFLN)** — lithium niobate on an insulator substrate — which dramatically reduces Vπ (to ~1–2 V), pushes bandwidth beyond 100 GHz, and shrinks the device, enabling compact, low-power, very-high-speed modulators. TFLN startups (HyperLight, and others) and established players are racing to commercialize it for 800G/1.6T coherent and direct-detect applications.

### Electro-Absorption Modulators

The **Electro-Absorption Modulator (EAM)** is an InP-based device that changes its **absorption** (rather than phase) under an applied field, via the Franz-Keldysh effect or quantum-confined Stark effect. EAMs are compact and have low Vπ, and crucially they can be **monolithically integrated with a DFB laser** to form an **EML (Electro-absorption Modulated Laser)** — a single compact device combining source and modulator. EMLs are the dominant transmitter in **400G-DR4 and 400G-FR4** datacenter transceivers, with bandwidths reaching ~100 GHz. The EML is one of the most important and highest-volume optical components in the datacenter, and its supply (dominated by a few makers — Lumentum, Coherent) is a key element of the transceiver supply chain (File 21, File 23).

### Silicon Photonics Modulators

**Silicon photonics (SiPh)** builds optical components in silicon waveguides using CMOS-compatible processes, promising the cost and integration advantages of the semiconductor industry. The catch is that silicon lacks a strong linear electro-optic (Pockels) effect, so SiPh modulators rely on the weaker **plasma-dispersion effect** (changing the free-carrier concentration in a PN junction to change the refractive index). This yields a relatively high Vπ·L (~2 V·cm) and limits modulation bandwidth to ~50 GHz, adequate for many datacenter applications but challenged at the highest speeds. A common enhancement integrates a **germanium EAM** (Ge absorber on a Si waveguide) for faster modulation (>100 GHz). Silicon photonics is championed by Intel, Cisco/Acacia, Coherent, and others, and it is central to co-packaged optics (File 13) because it allows dense integration of many optical channels on a chip — its lack of an efficient on-chip laser (silicon's indirect bandgap) being the principal remaining challenge, solved by hybrid III-V integration.

### IQ Modulators for Coherent

**Coherent** transmission requires modulating both the amplitude and phase of the optical field, and on both polarizations. This is done with a **dual-polarization IQ (DP-IQ) modulator**: two MZMs arranged in quadrature (90° apart) generate the in-phase and quadrature components for one polarization, the structure is duplicated for the orthogonal polarization, and the two are combined via a polarization beam combiner. This enables the rich modulation formats of coherent transmission — QPSK, 16QAM, 64QAM, and beyond (File 10) — packing many bits per symbol onto the optical carrier.

## Optical Receivers

### Direct-Detection Receivers

The simplest receiver is a **PIN photodiode** (a p-i-n junction, typically InGaAs for the 900–1650 nm range), which converts optical power directly to electrical current with a responsivity around 0.9 A/W at 1550 nm and bandwidths to 100 GHz. PIN-based direct detection — the receiver senses only optical *power*, discarding phase — is used in all the short-reach datacenter transceivers (SR, DR, FR, LR). The **Avalanche Photodiode (APD)** adds internal multiplication gain (10–40×) for higher sensitivity, useful in access PON and long-reach links where receiver sensitivity is critical, at the cost of more noise and complexity.

### Coherent Receivers

The **coherent receiver** is far more sophisticated and far more capable. It mixes the incoming signal with a **local oscillator (LO)** laser in a **90° optical hybrid**, and **balanced photodetectors** (four pairs for dual-polarization) recover the full complex optical field — both amplitude and phase, on both polarizations. This complete field recovery is what enables the DSP to digitally undo chromatic dispersion, track and compensate PMD and polarization rotation, and demodulate high-order QAM. Coherent detection provides a **10–15 dB sensitivity advantage** over direct detection and is the foundation of all long-haul and DCI transmission. Modern **Integrated Coherent Receivers (ICR)** monolithically integrate the 90° hybrid, the four balanced photodetectors, and trans-impedance amplifiers on an InP chip with bandwidths over 100 GHz for 800G coherent — a marvel of photonic integration (File 10).

## WDM — Wavelength Division Multiplexing

The defining technique of high-capacity optical transmission is **Wavelength Division Multiplexing (WDM)**: carrying many independent data channels on a single fiber, each on a different wavelength (color) of light, multiplexed together at the transmitter and separated at the receiver. WDM is what lets a single fiber carry tens of terabits per second.

### CWDM

**Coarse WDM (CWDM, ITU-T G.694.2)** uses wide **20 nm channel spacing** across a broad range (1271–1611 nm, up to 18 channels). The wide spacing means the lasers need no temperature control (uncooled DFB lasers suffice, since their wavelength can drift within the wide channel), making CWDM cheap. Datacenter **100G-LR4 and 400G-FR4** use a 4-channel scheme (often called CWDM4, around 1271/1291/1311/1331 nm, though strictly this tighter spacing is LAN-WDM) to carry four lanes on one fiber pair. CWDM's low cost suits short/medium reach where the channel count need not be high.

### DWDM

**Dense WDM (DWDM, ITU-T G.694.1)** packs channels tightly — **50 GHz or 100 GHz spacing** (100 GHz ≈ 0.8 nm at 1550 nm) — fitting ~96 channels across the C-band at 50 GHz, and 160+ with C+L-band. The tight spacing demands **wavelength-stable, temperature-controlled lasers** (cooled DFB or external-cavity lasers, stable to better than ±1.5 GHz for a 50 GHz grid). DWDM is the technology of long-haul, metro, and DCI transport.

DWDM system **capacity** has scaled enormously with coherent modulation: 100G PM-QPSK × 96 channels ≈ 9.6 Tbps per fiber pair; 400G PM-16QAM × 96 channels ≈ 38.4 Tbps; 800G PM-64QAM (fewer channels, C-band only) ≈ 64 Tbps; approaching the ~100 Tbps nonlinear-Shannon ceiling for C-band SMF. **Flex-grid DWDM** (a G.694.1 amendment) abandons the fixed grid for variable channel widths in 12.5 GHz increments, allocating just enough spectrum to each channel based on its baud rate and modulation — letting a **Bandwidth-Variable Transponder (BVT)** adapt its modulation, baud rate, and FEC to the available optical reach and squeeze maximum capacity from the spectrum.

## Conclusion

Optical networking is where the datacenter confronts fundamental physics, and where engineering ingenuity has repeatedly extended the apparent limits of glass and light. From the coincidence of minimum fiber loss and EDFA gain in the C-band, to the electronic compensation of dispersion and PMD in coherent DSP, to the packing of dozens of wavelengths into a single fiber by DWDM, the field has compounded physical understanding into ever-greater capacity and reach. The components surveyed here — the fibers, amplifiers, modulators, and detectors — are the building blocks of every optical link in the datacenter and beyond, from the 850 nm VCSEL spanning a few meters between racks to the coherent submarine system spanning an ocean. The chapters that follow build directly on these foundations: File 10 develops coherent optical transmission and the pluggable transceiver ecosystem in full; File 11 covers optical switching and ROADMs; File 12 covers datacenter interconnect and submarine systems; and File 13 covers the co-packaged-optics revolution that will pull these optical components inside the switch and the accelerator package itself. The physics established here is the unchanging substrate beneath all of it.
