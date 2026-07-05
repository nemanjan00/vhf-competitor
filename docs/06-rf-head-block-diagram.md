# 06 — RF head block diagram (v0)

First integration of decisions #6–#22 into one picture of the masthead
unit. Gain/level numbers are deliberately absent — the **level plan**
(cumulative gain / NF / IP3 per stage) is the next artifact and will be
built over this topology. Blocks marked ⚠ are open items.

## Signal path

```mermaid
flowchart TB
    ANT[Antenna port]

    subgraph FE[Front end]
        HPBPF["High-power BPF 144–146<br/>50 W rated, common TX/RX path<br/>(low-loss cavity — its insertion loss<br/>adds directly to system NF and ERP)"]
        TR[T/R relay + RX limiter]
        PRESEL["RX preselector BPF<br/>(2nd stage, RX-only path)"]
        ATT["Step attenuator 0/10/20 dB<br/>gain STATE, not AGC —<br/>software/operator set, changes rarely"]
        LNA[LNA]
    end

    subgraph RX[RX chain → IF 4–6 MHz]
        IMG["Image filter BPF<br/>(kills 134–136 image)"]
        RXMIX["RX mixer (level-17 brick)"]
        DIPLEX[Diplexer / termination]
        IFAMP[IF amp]
        AAF[Anti-alias BPF 4–6 MHz]
        ADC1[ADC ch 1<br/>16-bit class, ~16 Msps]
    end

    subgraph TXC[TX chain — zero-IF IQ]
        DACS[I/Q DAC pair]
        IQMOD[Quadrature modulator]
        TXBPF[TX BPF]
        DRV[Driver]
        PA["PA ~50 W ⚠<br/>(implied by BPF rating — confirm Q3)"]
        LPF[TX LPF / harmonic filter]
        COUP[Directional coupler]
    end

    subgraph FB[PureSignal feedback]
        FBATT[Calibrated attenuator]
        FBMIX["Feedback mixer<br/>(same 140 MHz LO)"]
        FBAAF[BPF 4–6 MHz]
        ADC2[ADC ch 2<br/>same digitizer]
    end

    ANT --- HPBPF --- TR
    TR --> PRESEL --> ATT --> LNA --> IMG --> RXMIX --> DIPLEX --> IFAMP --> AAF --> ADC1
    DACS --> IQMOD --> TXBPF --> DRV --> PA --> LPF --> COUP --> TR
    COUP --> FBATT --> FBMIX --> FBAAF --> ADC2
```

## LO / reference block

```mermaid
flowchart TB
    GPSANT[GPS antenna] --> GPSDO[GPSDO<br/>10 MHz + PPS]
    subgraph LO1["RX/FB LO — 140 MHz"]
        XO1[140 MHz overtone crystal<br/>TPU-isolated + mass]
        PLL1["clean-up PLL ÷14<br/>(~10–100 Hz loop ⚠ crossover TBD)"]
        PLL1 -.-> XO1
    end
    subgraph LO2["TX LO — ~144 MHz ⚠ exact parking freq TBD"]
        XO2[overtone crystal #2<br/>same mounting]
        PLL2[clean-up PLL]
        PLL2 -.-> XO2
    end
    subgraph CLK["Converter clocks"]
        CKPLL["ADC/DAC clock synth from 10 MHz<br/>⚠ jitter budget must match 16-bit"]
    end
    GPSDO --> PLL1
    GPSDO --> PLL2
    GPSDO --> CKPLL
    GPSDO -->|PPS| FPGA2
    XO1 ==> RXMIX2[→ RX mixer]
    XO1 ==> FBMIX2[→ FB mixer]
    XO2 ==> IQMOD2[→ IQ modulator]
    CKPLL ==> ADCs[→ ADC ch1/ch2, DACs]
    FPGA2[→ framer FPGA<br/>PPS sample-anchor]
```

## Digital / control / power

```mermaid
flowchart TB
    subgraph DIG[Digital]
        FPGA["Framer FPGA — deliberately dumb:<br/>ADC ch1+ch2 → Ethernet frames<br/>(sample counter + PPS anchor)<br/>TX IQ frames → DACs<br/>NTP server (GPS-disciplined)"]
        PHY["GbE PHY / SFP ⚠ copper vs fiber (Q3a)"]
        FPGA --- PHY
    end
    subgraph SAFE[Safety — independent of network]
        SEQ["Hardware sequencer:<br/>preamp → TR → PA order,<br/>watchdog → RX-safe on link loss"]
        KEY[Local keyer<br/>CW timing at the head]
        IMU[IMU<br/>vibration health/feed-forward]
    end
    subgraph PWR["Power ⚠ PoE vs hybrid (Q3)"]
        PD[PoE PD / DC in]
        PRECH[Precharge + inrush limit]
        SCAP[Supercap bank + balancer]
        DCDC[DC-DC rails<br/>filtered per bulkhead policy]
        PD --> PRECH --> SCAP --> DCDC
    end
    PHY ===|point-to-point to GS| GS[Ground station]
    SEQ --- FPGA
    IMU --- FPGA
```

## Block inventory (surplus shopping classes)

| Block | Candidate class | Note |
| --- | --- | --- |
| High-power BPF | 50 W-rated low-loss cavity, common path | first block after the antenna; passes TX, protects RX, cleans both directions |
| T/R relay | **low-loss coax relay**, ≥ 50 W | sequenced, not RF-sensed; its loss also counts twice (NF + ERP) |
| Preselector | telecom/PMR cavity or helical, retuned to 144–146 | sets out-of-band survival |
| Step attenuator | relay/PIN switched pads | overload insurance; engaged NF penalty is acceptable exactly when the band is loud |
| LNA | PGA-103/PSA4-class or Mini-Circuits brick | NF set here for the whole station |
| Image/IF/AA filters | SBP/SLP bricks + custom LC where needed | image reject + octave-clean IF |
| RX/FB mixers | ZFM/ZX05 level-17 | measured before trust |
| IQ modulator | eval-board or brick modulator | LO leak/image handled by #21 loop |
| PA | ⚠ surplus pallet, ~50 W class (confirm Q3) | run hot, DPD-linearized |
| Coupler | directional coupler, known coupling factor | calibration path for DPD |
| GPSDO | Thunderbolt-class surplus | station time+frequency root |
| Crystals ×2 | custom overtone order | the one genuinely custom part |
| ADC/DAC + FPGA | ⚠ platform choice (#8) | now "digitizer + framer", not full SDR |

## Gain policy: no AGC in hardware

The RX chain runs **fixed gain** — no signal-driven gain control anywhere
ahead of the ADC. The 16-bit budget (see 02) exists precisely so the
strongest expected in-band signal fits under ADC full scale while the
noise floor stays comfortably above the quantization floor; gain is a
*level-plan constant*, not a control loop. What replaces AGC, in three
tiers:

1. **Fixed gain, planned** — the level plan places the antenna-referred
   clip point above the strongest survivable neighbor.
2. **Gain state, not gain control** — the step attenuator: switched by
   software or operator for gross regime changes (kW neighbor 500 m away
   vs quiet band), changing maybe twice a contest. Deliberately *not*
   signal-driven — no pumping, no IMD-vs-desense breathing, and every
   recorded sample remains absolutely calibrated as long as the state is
   logged in the stream metadata (it must be).
3. **All fast level management is DSP at the GS** — per-slice digital AGC
   for operator audio comfort, where it can be per-receiver, mode-aware,
   and instantly re-tunable. The waterfall and the recording always see
   the raw, un-AGC'd band.

## Open items blocking the level plan

1. TX power target (Q3) → PA, T/R, coupler, attenuator, supply sizing.
2. Platform (#8) → actual ADC part → full-scale input level the whole RX
   chain must be planned toward.
3. TX LO parking frequency (in-band leak management, #22).
4. Clean-up loop crossover measurement (bench, once crystals exist).
