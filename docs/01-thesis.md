# 01 — Project thesis

## The problem

In a VHF contest, the scarce resource is not RF performance in the abstract —
it is **operator time between QSOs**. A conventional station (single VFO,
narrow-band receiver, no spectrum awareness) forces the operator to serially
scan a mostly-empty band to find the next contact. On VHF, where activity is
sparse and bursty, search time dominates everything else.

Modern SDR technique solves this — wideband waterfall, multiple simultaneous
receiver slices, skimmers feeding a band map, click-to-QSY — but no
off-the-shelf station packages it *for VHF contesting*. Retrofitting an SDR
tap onto an existing rig helps, but compromises the tap point, the sequencing,
and the integration.

## The approach: clean sheet, surplus lab blocks

Building from scratch is normally the expensive option. It stops being one
when the analog signal path is assembled from **used, lab-grade,
connectorized RF blocks**:

| Block class | Surplus source | Why it wins |
| --- | --- | --- |
| Mixers, amps, splitters, switches, attenuators, filters | Mini-Circuits connectorized bricks (ZFM/ZX05, ZFL/ZX60, ZFSWA, SLP/SBP…) | 10–30 % of new price; passive bricks are essentially ageless; conservative published specs |
| Frequency references | Telecom OCXOs, GPSDOs (Thunderbolt class) | Phase noise / stability no amateur rig ships with, for tens of euros — and LO purity is *the* spec that decides VHF contest RX performance |
| Preselection, isolators, PIN switches | Base-station / PMR teardowns | Cavity and helical filters near the amateur VHF/UHF bands exist as surplus |
| Test equipment | Used HP / R&S / Rigol analyzers, NanoVNA/TinySA for portable | Same arbitrage applied to the gear needed to verify all of the above |

The result is a **modular, independently testable, upgradeable** radio: every
block has connectors, every block can be measured on the bench, and any block
(the LO, say) can be upgraded later without touching the rest of the chain.
No monolithic rig offers that.

## What the approach costs (accepted, eyes open)

1. **Every used block must be measured before it is trusted.** Mixer diodes
   get blown, connectors wear, amps drift. The test bench is the first
   "module" of the project. Per-block measured-vs-datasheet records are part
   of this repo.
2. **Interconnect is a first-class design element.** A brick radio lives or
   dies on its level plan and its leakage budget. At −140 dBm sensitivity, LO
   leaking around a chain of jumpered bricks is a real failure mode. The
   cumulative gain / NF / IP3 spreadsheet is the central design artifact, not
   an afterthought.
3. **Three things bricks cannot provide:**
   - the **digitizer** (ADC/FPGA/DAC) — a proven SDR platform is bought, not
     built (candidates in [02-architecture-options.md](02-architecture-options.md));
   - the **power amplifier**;
   - the **software** — which is where the actual competitive gap lives, and
     where the innovation budget of this project is spent.

## Target feature set ("contest TRX done right")

- Full-band waterfall, permanently visible.
- 2–4 independent receiver slices: hold the run frequency while a second
  slice scans; the multi-VFO problem dissolves into DDC channels.
- CW/SSB skimming feeding an integrated band map; one-click QSY.
- GPSDO-disciplined frequency reference throughout.
- Sequencer (preamp / PA / T-R) designed in, not bolted on.
- First-class integration with contest loggers (audio, CAT, band-map
  spots).
