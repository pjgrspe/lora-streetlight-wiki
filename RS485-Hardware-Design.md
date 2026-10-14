# RS485 Hardware Design

The server link leaves the cabinet: different building, different ground reference, possibly a shared conduit with 230 V and the luminaire wiring. **Isolate it.**

## Transceiver

| Part | Isolation | Rate | ESD | Note |
|---|---|---|---|---|
| **ADM2587EBRWZ** | 2.5 kV rms, **signal + integrated isolated DC-DC** | 500 kbps | ±15 kV | ★ **Recommended.** Slew-limited → much better EMI. One IC, no external isolated supply |
| ADM2582EBRWZ | same | 16 Mbps | ±15 kV | Same family; ⚠ has a documented fail-safe biasing glitch, but only above ~1 Mbps |
| SN65HVD1782 | none | 1 Mbps | ±16 kV | Best *non*-isolated choice; ±70 V bus fault protection, <$1.50 |
| MAX485 / SP3485 | none | — | low | Bench-grade. **MAX485 is 5 V-only** — needs level shifting from a 3.3 V STM32WL |

Isolation costs ~$8–12. That is the price of not chasing ground loops in the field. Cheaper discrete route if BOM-constrained: `Si8621 + SN65HVD1782 + B0505S-1WR3`, ~$3.

## Termination and fail-safe bias

- **Termination:** 120 Ω across A/B at **the two physical ends only**. Make it **jumper-selectable** — the master gateway may or may not be an end node. ⚠ The Modbus serial spec specifies 150 Ω / 0.5 W for the 2-wire case; 120 Ω matching cable Z₀ is the more common industrial practice. Verify against the spec before finalising.
- **Fail-safe bias** at **exactly one node** — the master. An idle undriven bus floats, and receiver thresholds are only ±200 mV.

With two 120 Ω terminations (60 Ω equivalent) on a 5 V rail, `V_diff = 5 × 60 / (60 + 2·R_b)`:

| R_b | V_diff | Verdict |
|---|---|---|
| 390 Ω | 358 mV | Strong, heavy DC load |
| **560 Ω** | **254 mV** | ★ Classic value; matches the spec's 450–650 Ω range |
| 650 Ω | 220 mV | At the edge, no margin |
| 1 kΩ | 145 mV | **Fails** — below threshold |

> **Common bug: porting a 560 Ω / 5 V bias network to a 3.3 V design gives an unreliable idle state.** At 3.3 V you need **300–390 Ω**. Use the ADM2587E's isolated 5 V secondary for the bias network instead.

Modern "true fail-safe" receivers (offset thresholds) hold RX high on an idle/open bus without external bias. **Fit external bias anyway** — true-fail-safe protects against an *open* bus, not a bus idling with reflections or partially driven.

## DE/RE control — use the hardware, know its limit

STM32 USARTs implement Driver Enable in hardware: `CR3.DEM` enables the DE pin, `CR3.DEP` sets polarity, `CR1.DEAT` is assertion time and `CR1.DEDT` de-assertion time, **in units of 1/8 or 1/16 of a bit**.

**The critical limit:** 5 bits × 1/16 bit = max **~1.94 bit times** of de-assertion delay — 202 µs at 9600 baud. Modbus RTU needs **3.5 character times (≈4.01 ms at 9600 8N1)** of inter-frame silence.

> `DEDT` handles the *transceiver* turnaround. It cannot enforce the Modbus 3.5T gap. **That must be a software timer.**

Start with `DEAT = 8` (0.5 bit), `DEDT = 16` (1 bit). Tie `RE̅` to `DE` so you never receive your own echo — **or better, leave RE always enabled and use the echo as a transmit-integrity check**: compare echoed bytes against what you sent, and a mismatch means bus contention or a driver fault. A genuinely useful diagnostic counter.

## Timing rule you will get wrong

The Modbus serial spec fixes t1.5/t3.5 from character time **only for baud ≤ 19200**. Above that they are **fixed absolute values: t1.5 = 750 µs, t3.5 = 1.750 ms**.

Compute t3.5 from character time at 115200 and you get 334 µs — wrong, and it will cause frame-merge failures against a compliant master.

Budget at 9600 8N1: 1 char = 1.146 ms, t1.5 = 1.72 ms, t3.5 = 4.01 ms, DE de-assert = 104 µs.

## Protection, cable, grounding

- **TVS:** `SM712` — a 600 W asymmetrical array purpose-built for RS-485's −7 V/+12 V common-mode window. ±15 kV air, 40 A EFT, but only **12–19 A of 8/20 µs surge**.
- **For real outdoor surge, 12 A is not enough.** Two-stage: GDT (90 V pair) to isolated ground → series 10 Ω PTC/fusible resistors → SM712 → transceiver.
- **Cable:** shielded twisted pair, Z₀ = 120 Ω (Belden 3105A/9841, AWG 22–24). Daisy-chain only — **no stars, no stubs >0.3 m.**
- **Shield:** to PE at **one end only** (the server end), or at the far end through a 100 nF/1 kV cap to give HF a return without a DC loop.
- Bring the RS-485 **common (SC) through as a third conductor**, referenced to the transceiver's isolated ground via a **100 Ω series resistor**. This keeps you inside the common-mode window without creating a fault current path.
- **Distance/baud:** rule of thumb is a bit-rate × distance product of ~10⁸ baud·m. 9600 → 1000 m+, 19200 → 1000 m, 115200 → ~800 m.

**Recommended default: 19200 8E1** (Modbus's own default), with 9600 and 115200 selectable. Parity gives a cheap second error check on top of CRC16.

## Related

- [[Modbus-Register-Map-Design]]
- [[Firmware-Architecture]]
- [[Enclosure-Power-And-Surge]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. ADM2582E/2587E datasheet: https://www.analog.com/media/en/technical-documentation/data-sheets/adm2582e-2587e.pdf · TI fail-safe biasing: https://www.ti.com/lit/pdf/ssztak0 · Littelfuse SM712 · ADI staged surge circuit: https://www.analog.com/en/resources/analog-dialogue/articles/safeguard-your-rs-485-communication-networks.html · Modbus over Serial Line V1.02
