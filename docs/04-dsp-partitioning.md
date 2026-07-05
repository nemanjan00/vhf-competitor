# 04 — RX DSP partitioning: what streams down the Ethernet

## Decided part

The RX chain **digitizes the entire 2 m allocation at once** (2 MHz span,
144–146 MHz, R1 assumed). No tuning at the digitizer level — the full band
exists as samples inside the FPGA at all times. This is what makes the
full-band waterfall and N independent receivers structural properties of
the radio instead of features.

## Open part

What crosses the network to the operator PC:

### Option 1 — Stream the whole band, DSP on the host

```mermaid
flowchart LR
    ADC[ADC 2 MHz span] --> FPGA[FPGA<br/>decimate + pack only]
    FPGA -->|full-band IQ<br//>~100–200 Mbit/s| HOST[Host PC<br/>all DSP: slices, demod,<br/>waterfall, skimmer]
```

- Bandwidth is a non-issue on GbE: 2 MHz span × 2 × 16 bit ≈ **64 Mbit/s**
  raw (≈ 96 Mbit/s at 24-bit). The full band fits in under 10 % of the link.
  **The fork is therefore about latency and robustness, not throughput.**
- Maximum flexibility: new demodulators, skimmers, recording (full-band
  contest recording = replay/audit for free) are host software changes.
- Latency is host-OS-dependent (buffering, GC, scheduler); fine for SSB
  search-and-pounce, marginal for comfortable CW sidetone/QSK, and the
  radio is useless if the host hiccups mid-QSO.

### Option 2 — DSP on the FPGA, stream N VFOs

```mermaid
flowchart LR
    ADC[ADC 2 MHz span] --> FPGA[FPGA<br/>N × DDC + filter<br/>+ demod]
    FPGA -->|N narrow slices<br/>or audio streams| HOST[Host PC<br/>UI: waterfall? band map, logger]
```

- Deterministic, low latency — demod happens meters from the antenna;
  network carries audio/thin IQ. CW sidetone and fast T/R stop depending on
  host scheduling.
- Radio is self-sufficient: a dumb client (or a failure-degraded host)
  can still operate one slice.
- Rigid: every new mode/decoder is FPGA work; the full-band waterfall
  still needs *some* wideband product streamed anyway (see below).

### Option 3 — Hybrid (the shape most mature SDR protocols converge on)

```mermaid
flowchart LR
    ADC[ADC 2 MHz span] --> FPGA[FPGA]
    FPGA -->|"N × DDC slices (IQ)<br/>low-latency path"| HOST[Host PC]
    FPGA -->|"wideband product:<br/>full-band IQ or FFT/waterfall frames"| HOST
    HOST --> UI[slices → operating RX<br/>wideband → waterfall + skimmer + band map]
```

- Hardware DDC slices for *operating* (latency-critical), plus a wideband
  stream for *awareness* (waterfall, skimmer, recording — latency-tolerant).
- This is essentially the openHPSDR/Hermes model, which matters for
  decision #8: choosing such a platform gets Option 3's gateware for free
  instead of writing DDC chains from scratch.
- Open sub-question: wideband path as raw IQ (host does FFT, enables
  recording/replay) vs FPGA-side FFT frames (cheaper on the host, loses
  raw recording).

## Considerations to settle it

1. **Latency budget** — the real requirement behind Option 2. Needs a
   number: is comfortable CW QSK/sidetone (< ~10–20 ms round trip) a hard
   requirement, or is TX sidetone generated locally at the head anyway
   (in which case RX-path latency tolerances relax a lot)?
2. **Failure philosophy** — must the radio be operable with a degraded/
   generic client, or is the full custom host stack always present?
3. **Gateware budget** — custom FPGA work is the most expensive
   engineering in the project; leaning on an existing platform's DDC
   gateware (Option 3 via openHPSDR-class hardware) buys most of Option 2's
   benefit for near-zero cost.
4. **Full-band recording** — if contest replay/audit is wanted (it's
   cheap and very useful for post-contest analysis and skimmer tuning),
   raw wideband IQ must reach the host or head storage regardless.

## Direction under consideration: three-tier with a ground station

A refinement of Option 3 that moves the latency-critical DSP off the
operator PC **without** paying the custom-gateware cost of Option 2: insert
a dedicated **ground station** — a headless compute box at the base of the
mast — between the RF head and the operator position.

```mermaid
flowchart LR
    subgraph MAST[Masthead RF head]
        ADC[ADC 2 MHz span] --> FPGA[FPGA<br/>decimate + pack<br/>+ sequencer/keyer]
    end
    subgraph GS[Ground station — real-time station core]
        CHAN[Channelizer<br/>full band → N slices]
        NPU[NPU<br/>AI-assisted noise cancelling<br/>per slice]
        SKIM[Skimmer / band-map engine]
        REC[(Full-band IQ<br/>recording)]
        HID[Operator hardware I/O<br/>knobs · mic + PTT · paddle]
        SW{{Built-in Ethernet switch}}
        CHAN --> NPU
        CHAN --> SKIM
    end
    FPGA ==>|full-band IQ<br/>~100 Mbit/s<br/>dedicated point-to-point port| CHAN
    CHAN --> REC
    NPU --> SW
    SKIM --> SW
    SW ---|UI · audio · spots| T1[Terminal: laptop]
    SW ---|UI · audio · spots| T2[Terminal: laptop 2 / multi-op]
```

Why the extra tier (the reasoning behind it):

- **The mast link becomes point-to-point.** RF head ↔ ground station is a
  dedicated segment with exactly two endpoints: no switch queues, no
  competing traffic, no bufferbloat — timing and flow control are
  *predictable by construction*, not by QoS configuration. The
  latency-sensitive stream (IQ down, TX/keying up) lives entirely on this
  segment. It also frees the protocol: raw Ethernet frames or dumb UDP at a
  fixed rate work fine when nothing can congest the wire, so the masthead
  FPGA's network stack stays trivial.
- **The operator link becomes ordinary.** Ground station ↔ operator PC
  carries audio, waterfall frames, and spots — all jitter-tolerant — over
  whatever LAN/Wi-Fi exists. The timing-critical and convenience domains
  are physically separated instead of sharing one network.
- **Weight (and heat) come off the mast.** Compute lives at ground level;
  the masthead unit is only RF + digitizer + sequencer, which shrinks the
  enclosure, the wind load, the thermal problem, and the PoE budget.

Division of labor:

- **Masthead FPGA does the minimum**: digitize, decimate, pack, plus the
  hard-real-time safety jobs (sequencer, local keyer). No DDC chains, no
  demod — gateware stays near-stock for whatever platform is chosen (#8).
- **Ground station does all signal processing** on general-purpose compute:
  polyphase channelization of the 2 MHz band (trivial for a modern CPU),
  demodulation, skimming, full-band recording, and **NPU-accelerated noise
  reduction per slice**. It runs headless with a real-time-tuned stack —
  deterministic latency without host-PC hiccups, meters from the head, on
  mains power (no PoE constraint on compute).
- **The ground station is also the station core, not just a DSP box.**
  Two further roles:
  - **It absorbs the hardware complexity of the operator position.** The
    physical human interface — tuning knobs, mic with PTT, CW paddle —
    plugs into the ground station directly. That keeps every
    latency-critical human input (PTT! paddle!) on the deterministic
    real-time box instead of traversing a laptop OS, USB stack, and a
    network hop. Terminals need *zero* special hardware.
  - **It carries the audio chain — at professional grade.** Because all
    station audio funnels through one box, it only has to be built well
    once: studio-class codec (ADC/DAC), balanced low-impedance mic input
    with proper preamp (and phantom power, so studio mics are usable —
    surplus broadcast/studio gear is the same arbitrage as the RF bricks),
    a real headphone amplifier per operator, and clean analog supply rails
    isolated from the digital section. Contest-specific wins fall out:
    per-ear audio routing (run frequency left, S&P slice right), consistent
    TX audio (one known mic chain into the modulator instead of N laptop
    soundcards), and RX/TX audio that never touches a consumer sound
    device.
  - **It has a built-in Ethernet switch** and is the network boundary: one
    dedicated port for the point-to-point mast segment, switched ports for
    terminals (laptops). Plugging in a second laptop = a second operator
    position — multi-op falls out of the architecture for free.
- **Terminals become pure software thin clients** — UI, logger, audio over
  the switched ports. Any laptop works, can crash without taking the radio
  down, satisfies the failure philosophy question (#2 above) by
  construction.

On the AI noise cancelling itself, notes for when this materializes:

- **Audio-domain NR** (RNNoise / DeepFilterNet class, per demodulated
  slice) is proven tech, runs in single-digit ms on small NPUs, and helps
  most on SSB voice — which is most of VHF contest QSOs. Low risk.
- **IQ/spectrum-domain denoising** (before demod, or waterfall
  enhancement for weak-trace spotting) is more novel and more interesting —
  potentially also steerable against *local* noise (the station's own PSU
  spurs are stationary and learnable). Higher risk, own research track.
- Contest angle worth flagging: the same NPU can score **skimming**
  (CW decode, maybe SSB callsign spotting via lightweight ASR) — arguably
  worth more points than NR polish. The full-band recording (see #4 above)
  doubles as training data collection from day one.

## Status

Digitize-whole-band: **decided** (decisions.md #15). Partitioning: **open**
(decisions.md #16) — current direction is the **three-tier ground-station
hybrid** above; NPU hardware choice and NR domain (audio vs IQ) are open
sub-questions (#17).
