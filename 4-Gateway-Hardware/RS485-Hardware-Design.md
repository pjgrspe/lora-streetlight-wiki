# RS485 Hardware Design

The RS485 link carries Modbus RTU between the Local Monitoring Server and the Master Gateway. It leaves the cabinet (different building, different ground reference, possibly shared conduit with 230 V), so **it must be isolated and surge-protected**.

## Key points

- **Isolated transceiver: ADM2587E** (signal isolation plus integrated isolated DC-DC, slew-limited). Isolation is the price of not chasing ground loops in the field.
- **120 Ω termination at the two physical ends only**, jumper-selectable. **Fail-safe bias at exactly one node** (the Master).
- **Bias values don't port between rails:** 560 Ω suits 5 V; at 3.3 V you need 300–390 Ω. Better: run the bias from the ADM2587E's isolated 5 V side.
- **Hardware DE timing can't enforce the Modbus 3.5-character gap.** That gap must be a software timer.
- **Default 19200 8E1**, with 9600 and 115200 selectable. Two-stage surge protection (GDT → series resistor → SM712) for outdoor runs.

## Transceiver

| Part | Isolation | Rate | ESD | Note |
|---|---|---|---|---|
| **ADM2587EBRWZ** | 2.5 kV rms, **signal + integrated isolated DC-DC** | 500 kbps | ±15 kV | ★ **Recommended.** Slew-limited, so much better EMI. One IC, no external isolated supply |
| ADM2582EBRWZ | same | 16 Mbps | ±15 kV | Same family; ⚠ documented fail-safe biasing glitch, but only above ~1 Mbps |
| SN65HVD1782 | none | 1 Mbps | ±16 kV | Best *non*-isolated choice; ±70 V bus fault protection, <$1.50 |
| MAX485 / SP3485 | none | — | low | Bench-grade. **MAX485 is 5 V-only** and needs level shifting from a 3.3 V STM32. Use a 3.3 V part (SP3485 / MAX3485) on the bench |

Isolation costs ~$8–12. Cheaper discrete route if BOM-constrained: `Si8621 + SN65HVD1782 + B0505S-1WR3`, ~$3.

## Termination and fail-safe bias

- **Termination:** 120 Ω across A/B at **the two physical ends only**, jumper-selectable, because the Master may or may not be an end node. ⚠ The Modbus serial spec specifies 150 Ω / 0.5 W for the 2-wire case; 120 Ω (matching cable Z₀) is the more common industrial practice. Verify before finalising.
- **Fail-safe bias** at **exactly one node**, the Master. An idle, undriven bus floats, and receiver thresholds are only ±200 mV.

With two 120 Ω terminations (60 Ω) on a 5 V rail, `V_diff = 5 × 60 / (60 + 2·R_b)`:

| R_b | V_diff | Verdict |
|---|---|---|
| 390 Ω | 358 mV | Strong, heavy DC load |
| **560 Ω** | **254 mV** | ★ Classic value; matches the spec's 450–650 Ω range |
| 650 Ω | 220 mV | At the edge, no margin |
| 1 kΩ | 145 mV | **Fails**: below threshold |

**Common bug:** porting a 560 Ω / 5 V bias network to a 3.3 V design gives an unreliable idle state. At 3.3 V use **300–390 Ω**, or use the ADM2587E's isolated 5 V secondary.

"True fail-safe" receivers hold RX high on an *open* bus, but not on a bus idling with reflections or partially driven. **Fit external bias anyway.**

## DE/RE control

STM32 USARTs drive the transceiver's Driver Enable in hardware: `CR3.DEM` enables it, `CR3.DEP` sets polarity, `CR1.DEAT` / `CR1.DEDT` set assertion/de-assertion time in 1/8 or 1/16 bit units.

**The limit:** maximum de-assertion delay is ~1.94 bit times (202 µs at 9600 baud). Modbus RTU needs **3.5 character times (≈4.01 ms at 9600 8N1)** of inter-frame silence. `DEDT` handles transceiver turnaround only; **the 3.5T gap must be a software timer** ([[Firmware-Architecture]]).

Start with `DEAT = 8` (0.5 bit) and `DEDT = 16` (1 bit). **Leave RE always enabled and use the echo as a transmit-integrity check:** a mismatch between sent and echoed bytes means bus contention or a driver fault, which makes a useful diagnostic counter.

## Timing rule that is easy to get wrong

The Modbus serial spec derives t1.5/t3.5 from character time **only for baud ≤ 19200**. Above that they are fixed: **t1.5 = 750 µs, t3.5 = 1.750 ms**. Computing t3.5 from character time at 115200 gives 334 µs, which causes frame-merge failures against a compliant master.

At 9600 8N1: 1 char = 1.146 ms, t1.5 = 1.72 ms, t3.5 = 4.01 ms, DE de-assert = 104 µs.

## Protection, cable, grounding

| Item | Recommendation |
|---|---|
| TVS | `SM712`, a 600 W asymmetrical array for RS-485's −7/+12 V window. ±15 kV air, 40 A EFT, but only 12–19 A of 8/20 µs surge |
| Outdoor surge | Two-stage: GDT (90 V pair) to isolated ground → series 10 Ω PTC/fusible resistors → SM712 → transceiver |
| Cable | Shielded twisted pair, Z₀ = 120 Ω (Belden 3105A/9841, AWG 22–24). Daisy-chain only; no stars, no stubs > 0.3 m |
| Shield | To PE at **one end only** (server end), or at the far end via 100 nF/1 kV for an HF return without a DC loop |
| Common | Bring RS-485 common through as a third conductor, to the isolated ground via a 100 Ω series resistor |
| Distance | ~10⁸ baud·m rule of thumb: 9600 → 1000 m+, 19200 → 1000 m, 115200 → ~800 m |

**Default 19200 8E1** (Modbus's own default). Parity adds a cheap second error check on top of CRC16.

## Related

- [[Modbus-Register-Map-Design]]
- [[Firmware-Architecture]]
- [[Enclosure-Power-And-Surge]]
- [[Local-Monitoring-Server-Design]]

## Sources

- Research synthesis, 2026-09-09. ADM2582E/2587E datasheet: https://www.analog.com/media/en/technical-documentation/data-sheets/adm2582e-2587e.pdf · TI fail-safe biasing: https://www.ti.com/lit/pdf/ssztak0 · Littelfuse SM712 · ADI staged surge circuit: https://www.analog.com/en/resources/analog-dialogue/articles/safeguard-your-rs-485-communication-networks.html · Modbus over Serial Line V1.02

## History

- 2026-09-09: Written.
- 2026-10-14: Restructured; STM32WL references made MCU-neutral (gateway MCU is now a general STM32, see [[STM32-Part-Selection]]).

Back to [[Index]]
