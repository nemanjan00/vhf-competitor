# 07 — Parts list (living document)

One row per block of [06-rf-head-block-diagram.md](06-rf-head-block-diagram.md).
Status: **candidate** → **chosen** → **ordered** → **measured** (per the
measure-before-trust doctrine; measurement records will link from here).
Prices are rough placeholders to be verified at order time.

## Reference / LO

| Block | Candidate | Status | Notes |
| --- | --- | --- | --- |
| GPSDO | **Leo Bodnar GPSDO** (10 MHz + PPS) | **chosen** | Architecture-aware pick: its weaknesses (synth spurs, far-out PN) fall outside the clean-up loop BW where the crystals rule; what we use — accuracy, close-in stability, PPS — is what it's good at. ~€150 new |
| GPSDO (alt) | Trimble Thunderbolt (surplus) | candidate | True OCXO, better close-in; aging fleet, 12 V supply, bigger. Fallback / second unit for the bench |
| RX LO crystal | 140.000 MHz overtone, custom order | candidate | The long-lead item — spec and order early. Quartz houses: Krystaly, QuartsLab/Klove, IQD custom |
| TX LO crystal | ~144 MHz (parking freq TBD, #22) | blocked | Order together with RX crystal once parking frequency is fixed |
| Clean-up PLL ×2 | discrete PD + loop filter, or ADF4002-class PD-only PLL | candidate | ADF4002 = integer-N phase detector, no fractional spurs, loop filter sets the ~10–100 Hz BW |
| ADC/DAC clock | synthesized from 10 MHz — part depends on platform choice (#8) | blocked | Jitter budget must match 16 bits (02) |

## RX chain

| Block | Candidate | Status | Notes |
| --- | --- | --- | --- |
| 50 W BPF | surplus telecom/PMR cavity retuned; new: DCI-145-2H class | candidate | Lowest-loss block matters most: loss counts twice (NF + ERP). Spec target calibrated on 4O3A XL (≈0.5 dB IL, 55–75 dB stop, 5 resonators) — that series itself rejected: HF-only, 4.5 kW/65 cm shack unit, far too heavy for a masthead |
| T/R relay | surplus coax relay (Dow-Key / Radiall / Transco), SMA/N | candidate | Low loss, high isolation, ≥ 50 W hot-switch-safe rating with sequencer |
| RX preselector | helical 3-pole 144–146 (Toko-style or surplus PMR) | candidate | RX-only path — power rating irrelevant, loss still matters (pre-LNA) |
| Step attenuator | relay-switched pads (Mini-Circuits ZX76-class or relay+fixed pads) | candidate | 0/10/20 dB states; state must be reported into stream metadata |
| LNA | ZX60-P103LN+ (PGA-103) or PSA4-5043+ eval | candidate | NF ~0.5–0.6 dB at 145 MHz, high IP3 for its class |
| RX mixer | Mini-Circuits **ZFM-2H-S+** class (level 17) | candidate | Used market plentiful; measure conversion loss + ports before trust |
| Diplexer | built LC (bench project) or Mini-Circuits splitter/diplexer parts | candidate | Proper mixer IF-port termination — small but performance-critical |
| IF amp | ZFL-500LN+ class | candidate | Modest gain, low NF at 4–6 MHz |
| Anti-alias BPF | SBP-5+ class + trimmed LC, or custom LC 4–6 MHz | candidate | The load-bearing filter of the single-ADC design — measured shape required |

## TX chain

| Block | Candidate | Status | Notes |
| --- | --- | --- | --- |
| I/Q DAC + IQ modulator | depends on platform (#8); discrete alt: AD9767 + ADL5375 eval boards | blocked | |
| Driver | ZHL-class brick or surplus pallet driver stage | candidate | Sized after PA choice |
| PA ~50 W | surplus VHF pallet (Mitsubishi RA60H1317M module class) | candidate | Run efficient, DPD-linearized; confirm 50 W target (Q3) |
| TX LPF | built LC Chebyshev, 7-pole | candidate | Harmonic spec is regulatory, measure |
| Directional coupler | surplus 20–30 dB coupler, known coupling factor | candidate | Feeds PureSignal path; calibration constant of the DPD system |
| Feedback attenuator | fixed SMA pads, calibrated | candidate | |
| Feedback mixer | second ZFM-2H-S+ (same 140 MHz LO) | candidate | Same class as RX mixer |

## Digitizer / framer platform (#8 — the big open)

| Candidate | For | Against |
| --- | --- | --- |
| **Red Pitaya SDR 122-16** (16-bit LTC2185 ADC, 2× 14-bit DAC, Zynq, GbE) | Checks nearly every box: true 16-bit ADC, two ADC inputs (RX + feedback), DAC pair for IQ TX, Ethernet-native, existing openHPSDR-compatible gateware **with PureSignal support** (Pavel Demin's projects) — the "near-stock gateware" plan holds | 14-bit DACs (fine: TX purity is DPD's job); verify input full-scale + clocking from external 10 MHz |
| Discrete: LTC2185 eval + framer FPGA board | Exactly and only what we need | Custom FPGA work — violates the no-custom-gateware doctrine |
| Used HF SDR (single RX only) | Cheap phase-1 RX-only start | No TX path, no feedback channel — phase 1 stopgap at best |

## Masthead / infrastructure

| Block | Candidate | Status | Notes |
| --- | --- | --- | --- |
| Enclosure | IP-rated die-cast alu, PA wall as heatsink, finned | candidate | Doc 03 thermal design |
| Supercap bank + precharge + balancer | Maxwell/Eaton surplus supercaps + own board | candidate | Sized after Q3 confirmation |
| PoE PD / DC input | 802.3bt PD module (if 50 W confirmed → PoE++ plausible) | candidate | Doc 03 power math |
| Sequencer / keyer MCU | small MCU board (own firmware — the one bit of embedded code we own) | candidate | Independent of network path |
| IMU | any drone-grade IMU breakout | candidate | Vibration characterization + permanent health sensor |
| GPS antenna | timing patch antenna, mast-top | candidate | |
| Jumpers | surplus Sucoflex/RG-402 per doc 01 policy | candidate | Measure loss + connector condition on arrival |

## Ground station (phase 2 — deferred)

Desktop PC serves as GS in phase 1 (doc 05). Rack chassis, NPU card,
studio audio interface, switch: listed when phase 2 opens.
