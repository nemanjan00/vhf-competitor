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

Remaining before closing: the combined platform decision (#8 + #22 TX) —
the digitizer, the PureSignal feedback channel, and the IQ TX path should
land on one coherent platform. A costed trade study with real surplus
prices is the next step.

## TX side

Deliberately deferred until the RX architecture is fixed; both options
imply a matching upconversion/exciter strategy (B gives it almost for free
— the same mixer topology run in reverse; A implies a TX DAC on the
platform).
