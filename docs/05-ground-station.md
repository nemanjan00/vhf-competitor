# 05 — Ground station

Consolidates the ground-station requirements accumulated in
[04-dsp-partitioning.md](04-dsp-partitioning.md) into a component-level
view. The GS is the **station core**: everything except RF and the thin
clients lives here.

## Accumulated requirements

| Domain | Requirement | Source |
| --- | --- | --- |
| DSP | Channelize full 2 MHz band → N slices; demod; skimmer; full-band IQ recording | 04 |
| ML | NPU for AI noise cancelling (audio-domain first); candidate later: skimming assist | 04 |
| Audio | Professional grade: balanced mic in + phantom, studio codec, headphone amp per operator, per-ear routing matrix | 04 |
| HID | Physical operator I/O terminates here: tuning knob(s), PTT, CW paddle | 04 |
| Network | Built-in Ethernet switch: 1 × dedicated point-to-point port to mast head, N × terminal ports | 04 |
| Timing | Real-time-tuned stack; PTT/paddle handled with deterministic latency | 04 |
| Acoustics | Sits on the operating desk next to an open mic → fanless or near-silent | 03 |
| Placement | Within arm's reach of operator (knobs, mic) | 04 / Q3f |
| Storage | Record the **entire 2 m band, full contest duration** (see sizing below) | 04 / this doc |

## Full-contest band recording

Recording the whole band for the whole contest is cheap enough that the
question inverts: there is no reason *not* to.

Sizing at 2.5 Msps complex (covers the 2 MHz allocation with filter
margin):

| Bit depth | Data rate | 24 h contest | 48 h |
| --- | --- | --- | --- |
| 2 × 8 bit | 5 MB/s | 432 GB | 864 GB |
| 2 × 16 bit | 10 MB/s | 864 GB | 1.7 TB |
| 2 × 24 bit | 15 MB/s | 1.3 TB | 2.6 TB |

**Requirement: minimum 48 h of the full 2 MHz band.** At full 16-bit depth
that is 1.7 TB — a single commodity 2 TB NVMe covers the minimum, and a
4 TB drive gives 48 h with margin plus room to keep the *previous*
contest while operating the current one. The ~10 MB/s write rate is
trivial (no RAID, no exotic filesystem — one drive, ring-buffer files).
Storage lives in the ground station, on the same box already receiving
the full-band stream.

What it buys, in rough order of contest value:

1. **Instant replay while operating** — re-listen to the exchange you
   half-copied *seconds ago* on any frequency, not just the one you were
   on. A "what did he say?" button is arguably the single highest-value
   UI feature this architecture enables.
2. **Post-contest audit** — resolve every questionable QSO in the log
   against what was actually on the air; log-checking disputes end.
3. **Training data** — labeled, real-condition IQ for the NPU noise
   cancelling and skimmer models, collected as a side effect of operating.
4. **Skimmer development loop** — replay the same contest through improved
   decoders and measure the improvement on ground truth.
5. **Missed-multiplier forensics** — after the contest, sweep the recording
   for stations that were workable but never found. Directly measures the
   thing this whole project exists to fix.

### Timestamping

The recording can be timestamped to a precision far beyond anything a
log needs, essentially for free: the ADC sample clock at the head is the
GPS-disciplined reference (decision #19), so **the sample counter itself is
a clock** — it never drifts against GPS. Anchor the counter to absolute
time once per second (PPS marks the sample on which the second begins) and
every one of the ~430 billion samples in a 48 h recording carries an
absolute GPS-traceable timestamp; at 2.5 Msps the raw granularity is
400 ns, and the PPS anchor keeps absolute error in the tens of ns.

Design obligation: the head→GS stream protocol must carry `(sample
counter, PPS anchor)` metadata, and the recorder must preserve it —
including across any dropped-packet gaps (gaps must consume counter space,
not silently concatenate).

What sub-µs timestamps unlock beyond tidy logs:

- **Cross-station correlation** — two or more stations recording the same
  contest with this architecture can align their IQ to sub-µs and do TDOA:
  passively localize interference, or study how the same signal arrived at
  two sites (aircraft scatter, ducting).
- **Propagation forensics** — meteor-scatter ping timing, Doppler tracks of
  aircraft reflections, es opening onset times — all measurable from the
  archive.
- **Exact log reconciliation** — every logged QSO maps to a sample range;
  replay is indexable by log entry.

## Convergence: the GS is a PC with a lot of PCIe

Walking the requirements table, every line maps onto a slot or header of
an ordinary PC platform:

| Requirement | PC realization |
| --- | --- |
| Channelization / demod / skimmers | x86 CPU — 2 MHz of DSP is light work for modern cores |
| NPU | M.2 or PCIe accelerator (Hailo-8 class), or GPU slot |
| 48 h+ IQ recording | M.2 NVMe (2–4 TB) |
| Mast point-to-point port | dedicated NIC port (or SFP NIC → fiber, Q3a) |
| Terminal switch | multi-port NIC + software bridge, or a small managed switch board in the chassis |
| Pro audio | PCIe/USB studio interface — an entire commodity market exists (balanced I/O, phantom, headphone amps) |
| HID (knobs, PTT, paddle) | USB, or a small microcontroller board for the timing-critical inputs |
| Real-time stack | Linux + RT tuning, CPU core pinning for the DSP path |

**Do the DSP filters need an FPGA? No.** The rule of thumb: FPGAs earn
their complexity when sample rates are high (hundreds of Msps) or when
latency must be cycle-deterministic. Neither applies below the head:

- The one place filtering happens at high rate — the ADC output, at the
  full converter rate — is inside the masthead SDR platform's FPGA, and
  that decimation-to-2.5-Msps gateware ships **stock** with any candidate
  platform (#8). We write none of it.
- Everything downstream runs at ≤ 2.5 Msps complex. A generously sized
  polyphase channelizer, all the per-slice FIRs, and every demodulator
  together amount to a few Gmul/s — a fraction of one AVX-capable core.
  Filters become arrays in memory: redesignable at runtime (per-mode
  shapes, adjustable skirts, experiment-friendly) instead of frozen in
  gateware.
- Latency from CPU block processing at these rates is sub-millisecond to
  a few ms with RT tuning — inside any budget Q3c can plausibly set. The
  only cycle-critical jobs (sequencer, keying) were already assigned to
  the head's FPGA/MCU for safety reasons, not throughput.

So the partition stays clean: **FPGA where the samples are fast (head,
stock gateware); CPU where the samples are slow (GS, our code)**. Custom
gateware remains a thing this project deliberately does not write.

No exotic hardware anywhere — the ground station is a **software product
plus system integration**, not a hardware development project. The only
custom physical element left is the front panel. This effectively answers
the compute-platform question (Q3d) in favor of x86 + PCIe peripherals;
the remaining constraint is **silence** (passive-cooling x86 builds and
near-silent big-fan builds are both proven territory — the CPU chosen for
DSP headroom must fit that thermal budget).

It also de-risks the whole tier: every component is commodity,
replaceable mid-contest from any computer shop, and upgradeable
independently — the same brick philosophy as the RF chain, in PCIe form.

## Form factor: 19″ rack server case, modified front panel *(considering)*

The idea: a standard rack server chassis, front panel replaced/machined
into the actual operator panel.

```mermaid
flowchart TB
    subgraph CASE[19″ rack chassis]
        subgraph FRONT[Modified front panel]
            KNOB[Tuning knob&#40;s&#41;]
            MICIN[XLR mic in]
            HP[Headphone jacks × N]
            PTTJ[PTT / paddle jacks]
            DISP[Status display?]
        end
        subgraph INSIDE[Internals]
            COMPUTE[Compute board<br/>SBC or mini-ITX x86]
            NPUM[NPU module<br/>M.2 / SoC-integrated]
            AUDIO[Audio board<br/>preamp · codec · HP amps]
            SWB[Ethernet switch board]
            PSU[Silent PSU<br/>linear or fanless]
        end
        subgraph REAR[Rear panel]
            MASTP[Mast port<br/>point-to-point &#40;or SFP&#41;]
            TERMP[Terminal ports × N]
            PWRIN[Mains in]
        end
    end
    KNOB & PTTJ --> COMPUTE
    MICIN --> AUDIO --> COMPUTE
    COMPUTE --> NPUM
    COMPUTE --> SWB
    SWB --> MASTP & TERMP
```

**Why it fits:**

- **Panel real estate is the scarce resource** for knobs, XLR, headphone
  jacks, PTT connectors — a 19″ front panel is exactly that, and a custom
  aluminum panel is a cheap, well-understood fabrication job (or even
  hand-drilled to start).
- **Same surplus arbitrage again**: used rack chassis are near-free, built
  like tanks, shielded by construction, with proper card mounting, PSU
  bays, and cable management.
- **Room to grow.** Compute board + NPU + audio board + switch + PSU with
  space left over; internal layout can separate the analog audio corner
  from switching supplies and digital boards (grounding/shielding
  discipline inside one box is decision-critical for the "pro audio"
  claim).
- **It looks like what it is** — a piece of station infrastructure, not a
  pile of SBCs. Standard rails if it ever moves into a rack; feet if it
  stays on the desk.
- **It buys into a whole ecosystem, not just a box.** 19″ is the shared
  standard of pro audio, telecom, and lab gear — so the station can grow
  as rack units in a **music-style desk rack**: the GS itself, a **linear
  PSU as its own rack unit** (the pro-audio answer to switching-supply
  noise in both the mic chain and the RF environment — and surplus rack
  linears are plentiful), and later whatever the station needs — patch
  panel, UPS/supercap unit, a spare-blocks drawer. Music racks specifically
  are made to live next to microphones: short depth, desk height, quiet by
  convention. Each function becomes a swappable rack unit — the same
  modularity philosophy as the SMA bricks, one level up.

**Obligations it brings:**

- **Server chassis assume screaming forced airflow; this build forbids
  it.** Component budget must fit passive/near-silent cooling: efficient
  compute (see NPU question), fanless PSU (or oversized ultra-quiet fan at
  low RPM), vented top over the warm zone. 2U is tight for passive; **3U–4U
  gives fin height and panel height** (a proper tuning knob wants a taller
  panel anyway).
- Depth: server cases are deep (600 mm+) for a desk — a short-depth
  (~350–450 mm) chassis or audio-gear-style case may fit the desk better
  while keeping the 19″ panel.
- Front-panel HID and audio wiring crosses the box — keep mic lines
  balanced and short, route away from the switch and PSU.

## Build phasing: the GS starts as a desktop

Because the ground station is defined by its **software and its network
role**, not its box, the GS can begin life as the existing desktop PC:

1. **Phase 1 — RF first, desktop as GS.** Build and bench the RF
   head; plug its point-to-point link into a desktop's spare NIC. The
   desktop runs the full GS stack (channelizer, waterfall, recorder,
   skimmer) — every line of which is the *real* software, developed
   against real signals from day one. Audio via any interface, HID via
   USB. Nothing is throwaway.
2. **Phase 2 — dedicated GS box.** When the rack chassis materializes, the
   software moves unchanged; the desktop's role shrinks back to being a
   terminal on the switched side. NPU, studio audio, front panel arrive as
   incremental PCIe/panel additions.
3. **Phase 3 — masthead deployment.** The bench-proven head goes up the
   mast; nothing else changes.

This ordering also front-loads the two highest-risk items (LO block, RF
integration) and defers the lowest-risk purchases (chassis, NPU, audio
gear) until the system already works.

## Open questions (queued in QUESTIONS.md)

- Compute + NPU platform (Q3d) — decides everything downstream of the
  motherboard tray.
- HID transport: USB devices vs GPIO encoders (Q3g).
- Chassis height/depth and desk vs under-desk rack placement.
