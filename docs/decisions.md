# Decision log

Lightweight record of design decisions. One line each; link out to a doc
when the reasoning deserves more space. Status: **decided** / **leaning** /
**open**.

| # | Decision | Status | Notes |
| --- | --- | --- | --- |
| 1 | Clean-sheet station, not a retrofit of an existing rig | **decided** | Frees the design to use surplus lab-grade blocks; see [01-thesis.md](01-thesis.md) |
| 2 | Analog signal path built from used connectorized RF blocks (Mini-Circuits class) + telecom surplus | **decided** | The cost/performance arbitrage that makes the project viable |
| 3 | Digitizer (ADC/FPGA/DAC) is a bought, proven SDR platform — not custom silicon/board design | **decided** | Innovation budget goes to RF integration + contest software |
| 4 | Diagrams in Mermaid, docs in Markdown, in-repo | **decided** | GitHub renders both natively |
| 5 | Every used block measured before trusted; records kept in-repo | **decided** | Test bench is the first module of the project |
| 6 | RX architecture: direct sampling vs brick superhet vs hybrid | **open — leaning B/C** | Trade study in [02-architecture-options.md](02-architecture-options.md) |
| 7 | Band plan: **2 m only** — the VHF ham band (144–146 MHz in IARU R1, assumed) | **decided** | Radio is purpose-built for VHF contests; no 70 cm/23 cm converters. Single band → single preselector/converter/PA chain, and the whole 2 MHz allocation fits one digitizer span |
| 8 | Digitizer platform choice | **open** | Depends on #6; candidates: HF 16-bit SDR class for B, Hermes/ANAN/RFSoC class for A |
| 9 | LO / reference strategy | **open** | GPSDO-disciplined OCXO assumed; synthesis chain (multiplied OCXO vs modern PLL, e.g. LMX2594-class) TBD |
| 10 | Contest software stack (build vs integrate with existing loggers/skimmers) | **open** | Likely integrate first (CAT/band-map protocols), build the glue |
| 11 | Power amplifier strategy | **open** | Deferred until TX architecture exists; PoE-vs-hybrid feed gates on target power |
| 12 | Station topology: radio is a **mast-mounted RF head** below the antenna; controlled over **Ethernet**; PoE if the power budget allows, else Ethernet + DC pair | **decided** | Feedline loss eliminated on both RX (NF) and TX (ERP); digitizer must be Ethernet-native; local hardware sequencer mandatory. See [03-station-topology.md](03-station-topology.md) |
| 13 | Masthead power: feed sized for **average** draw, **supercap bank** buffers TX peaks; all supply rails filtered (common-mode + feedthrough at bulkhead) | **decided** | Peaky SSB/CW duty cycle → thinner feed, no sag at voice peaks; precharge, balancing, and DC-DC to PA rail are design obligations. See [03-station-topology.md](03-station-topology.md) |
| 14 | Data link medium: copper Ethernet vs fiber + separate DC | **open** | Fiber kills surge and RFI coupling problems outright; surplus media converters/SFPs are cheap |
