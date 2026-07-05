# 02 — Architecture options (trade study, RX side)

The first fork in the design: where does the signal get digitized?
Everything else — parts list, level plan, LO strategy, digitizer choice —
follows from this.

Both options share the same antenna-side front end (preselector, LNA, T/R
switching, sequencer) and the same software layer. They differ in what sits
between the LNA and the DSP.

## Option A — Direct sampling

Digitize the whole band (or a large chunk of it) directly at VHF.

```mermaid
flowchart LR
    ANT[Antenna] --> TR[T/R switch<br/>+ limiter]
    TR --> PRESEL[Preselector<br/>cavity/helical BPF]
    PRESEL --> LNA[LNA]
    LNA --> AAF[Anti-alias /<br/>band filter]
    AAF --> ADC[ADC<br/>direct @ VHF]
    CLK[OCXO/GPSDO<br/>clock] --> ADC
    ADC --> FPGA[FPGA<br/>N × DDC slices]
    FPGA --> HOST[Host PC<br/>waterfall · skimmer · band map · logger]
```

**For:**
- Minimal analog chain — fewer blocks to source, measure, and shield.
- N simultaneous receivers fall out of the FPGA for free; the multi-VFO
  goal is trivially met.
- No mixer spurs, no image problem, no IF planning.
- "LO purity" becomes clock jitter — and a surplus OCXO/GPSDO clock chain
  can be excellent.

**Against:**
- The ADC platform choice dominates performance; the surplus-brick arbitrage
  barely applies to this option — most of the budget lands on the digitizer.
- Blocking/overload handled almost entirely by ADC dynamic range plus the
  preselector; less opportunity to distribute gain and filtering through a
  chain.
- High-performance direct-sampling platforms at VHF (16-bit ADC class,
  RFSoC) are the one part of the market where surplus is *not* cheap.

## Option B — Brick superhet down to an HF-class SDR (transverter topology)

Convert 144 MHz (etc.) to a low IF with surplus mixers and a lab-grade LO,
then digitize with a cheap, proven HF SDR.

```mermaid
flowchart LR
    ANT[Antenna] --> TR[T/R switch<br/>+ limiter]
    TR --> PRESEL[Preselector<br/>cavity/helical BPF]
    PRESEL --> LNA[LNA]
    LNA --> RFBPF[Image filter<br/>BPF]
    RFBPF --> MIX[Level-17<br/>diode mixer]
    LO[LO chain<br/>OCXO → synth → BPF/amp] --> MIX
    MIX --> DIPLEX[Diplexer /<br/>termination]
    DIPLEX --> IFAMP[IF amp]
    IFAMP --> IFBPF[IF BPF]
    IFBPF --> SDR[HF SDR<br/>e.g. 16-bit @ 0–30 MHz]
    REF[GPSDO 10 MHz] --> LO
    REF --> SDR
    SDR --> HOST[Host PC<br/>waterfall · skimmer · band map · logger]
```

**For:**
- Plays directly to the surplus thesis: mixers, LO blocks, IF amps, filters
  are exactly what the used market sells cheap.
- Gain, filtering, and dynamic range distributed across a chain the designer
  controls — classic contest-grade RX engineering is possible.
- The digitizer becomes a commodity HF SDR (well-understood, cheap,
  replaceable).
- Multiband later = one more converter front end per band into the same IF.

**Against:**
- Much more analog chain: image filtering, IF planning, diplexers, mixer
  termination, LO leakage management — every classic superhet problem is
  back on the table.
- Multi-RX is limited to the IF bandwidth the converter passes (typically
  fine: a whole 2 MHz band segment fits, but it must be designed for).
- The LO chain must be genuinely good, or the whole phase-noise argument
  collapses. (Mitigation: this is exactly the block the surplus market is
  best at.)

## Option C (hybrid, noted for completeness)

A wideband direct-sampling *spotting* receiver (Option A, built cheap —
even an RX-only SDR) alongside a brick superhet *primary* receiver
(Option B). The spotter only needs to find signals; the primary only needs
to work them. Decouples the performance requirements of the two jobs.

## Status

Open — see [decisions.md](decisions.md), but converging on **B**. Two
forces now point the same way:

1. The parts thesis (surplus mixers/LO/IF bricks are where the arbitrage
   lives) — the original argument.
2. A digitizer preference set in the RX requirements: **high resolution,
   low bandwidth**. The band is 2 MHz wide and never wider (#7, #15), so
   sample rate above a few Msps buys nothing — while every extra ADC bit
   buys blocking dynamic range in a strong-neighbor contest environment.
   A 16-bit-class ADC at low rate is exactly the commodity HF-SDR
   digitizer of Option B; wideband direct-sampling (Option A) spends its
   money on bandwidth we'd throw away. High-res low-BW converters can't
   digitize 144 MHz directly — the conversion stage isn't a compromise,
   it's what lets the best-fit ADC be used at all.

### How many bits are actually useful? (~16 — beyond that, diminishing returns)

Marketing bits ≠ delivered bits: a good "16-bit" Msps-class ADC delivers
~12.5–13.5 ENOB (~78–84 dB SNR at full scale). That sounds short of
contest-grade until **processing gain** is counted: the noise is spread
across the whole digitized span, and filtering 2.5 Msps down to a 500 Hz
CW slice recovers 10·log₁₀(2.5 M/500) ≈ **37 dB**. So a real 16-bit
converter yields ~**115 dB dynamic range in 500 Hz** — the same territory
as the best analog contest receivers ever built.

Why more bits stop paying:

- Above that level, **other limits dominate**: clock jitter (reciprocal
  mixing's digital twin), front-end IMD in the LNA/mixer, and the band's
  own noise floor arriving through the antenna. A 20-ENOB converter behind
  a chain that delivers 110 dB is buying resolution for signals the chain
  has already corrupted.
- True >16-ENOB converters at Msps rates essentially don't exist —
  "24-bit" parts at these speeds deliver nowhere near 24 (delta-sigma
  parts hit high ENOB only at audio-class rates).
- The last practical win of the 16-bit class is operational: enough
  headroom to run **fixed RF gain with no AGC ahead of the ADC** — the
  strongest neighbor just uses the top bits while the noise floor stays
  put. That property arrives at ~16 bits; more bits don't add a second
  such threshold.

Spec consequence: digitizer target is a **true 16-bit-class ADC (≥ ~12.5
ENOB at the IF), clocked from the GPSDO-disciplined reference with a
jitter budget matched to those bits** — clock quality is spent as
carefully as bit count, or the bits are fiction.

### LO implementation: crystal at the LO frequency, disciplined by PLL *(leaning)*

Refinement of decision #9. Two ways to make a fixed, GPSDO-traceable
~142 MHz LO (144–146 → 2–4 MHz IF):

1. **Multiply 10 MHz up** — inherits the reference's phase noise
   **+ 20·log₁₀(14.2) ≈ +23 dB** everywhere. Simple, but pays the
   multiplication tax at every offset.
2. **Run an actual VHF crystal oscillator at 142 MHz** (overtone crystal),
   **phase-locked to the 10 MHz GPSDO through a deliberately narrow
   loop**. Inside the loop bandwidth the PLL steers the crystal — accuracy
   and long-term drift become the GPSDO's. Outside the loop bandwidth the
   PLL is blind and the output is the **crystal's own phase noise at
   carrier frequency — no multiplication tax at all**.

```mermaid
flowchart LR
    GPSDO[GPSDO 10 MHz] --> PD[Phase detector<br/>÷N comparison]
    XO[142 MHz overtone<br/>crystal oscillator] --> PD
    PD -->|"narrow loop filter<br/>(~10–100 Hz BW)"| XO
    XO ==>|LO out: GPSDO accuracy,<br/>crystal purity| MIX[RX mixer]
```

The design subtlety that decides whether this wins: **inside** the loop
bandwidth the output still carries the reference's multiplied noise
(+23 dB); **outside** it carries the crystal's. So the loop bandwidth must
sit at the crossover where the multiplied-reference curve meets the
crystal's own curve — typically tens of Hz. Set it too wide and the loop
injects the multiplication tax back in; too narrow and the crystal wanders
thermally before the loop catches it. This crossover measurement is a
bench task once parts are in hand.

Practical notes:

- 142 MHz means a 5th/7th-overtone crystal — custom-ordered (quartz houses
  still cut these; also the standard transverter-LO crystal ecosystem,
  e.g. 116 MHz, shows the supply chain exists) or harvested.
- 142 MHz sits safely out-of-band; the image (138–140 MHz) falls to the
  preselector.
- **Microphonics are the new enemy**: VHF crystals are vibration-sensitive
  and this one lives *on a mast in the wind*. Mechanical mounting
  (mass, damping, orientation) is part of the LO design, and the narrow
  loop can't correct vibration-rate phase modulation — it's outside the
  loop BW by construction.
- The same architecture serves the TX IQ LO (mid/edge-band per #22) with a
  second crystal.

Remaining before closing: the combined platform decision (#8 + #22 TX) —
the digitizer, the PureSignal feedback channel, and the IQ TX path should
land on one coherent platform. A costed trade study with real surplus
prices is the next step.

## TX side

Deliberately deferred until the RX architecture is fixed; both options
imply a matching upconversion/exciter strategy (B gives it almost for free
— the same mixer topology run in reverse; A implies a TX DAC on the
platform).
