# vhf-competitor — design docs

A clean-sheet VHF contest transceiver station, designed around two observations:

1. **Contest operating technique has outrun contest radio architecture.** The
   single-VFO, knob-spinning workflow wastes most of its time *finding* the
   next QSO — exactly the part a wideband multi-receiver SDR with a band map
   makes nearly free.
2. **The used/surplus market is wildly asymmetric in the builder's favor.**
   Lab-grade connectorized RF blocks (Mini-Circuits bricks, telecom OCXOs and
   GPSDOs, base-station filters, surplus test gear) sell for a fraction of new
   price and outperform the front ends of off-the-shelf amateur rigs.

## Documents

| Doc | Contents |
| --- | --- |
| [01-thesis.md](01-thesis.md) | Why build instead of buy; what the surplus approach wins and what it costs |
| [02-architecture-options.md](02-architecture-options.md) | Candidate RX/TX architectures and the trade study between them |
| [decisions.md](decisions.md) | Decision log — settled and open |

## Conventions

- All docs are Markdown; diagrams are [Mermaid](https://mermaid.js.org/)
  (rendered natively by GitHub).
- Every used RF block that enters the system gets measured before it is
  trusted; measured-vs-datasheet records will live in this repo alongside the
  design.
- This repo is public. No credentials, no personal data, no scraped
  proprietary documentation.
