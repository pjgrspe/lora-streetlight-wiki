# DALI Telemetry And D4i

What per-light data the system can get from the luminaire drivers. The project uses **existing off-the-shelf DALI drivers** (decision 2026-09-13), so the telemetry must be **discovered per driver model**, and the design must work with only the basic status byte.

## Key points

- Parts 251/252/253 (asset data, energy, diagnostics) are **optional in DALI-2, mandatory in D4i**. With existing drivers, they may not exist.
- **The per-pole device probes capabilities at commissioning** and after a driver swap. In the v0.1 protocol it reports only a **`d4i_available` status bit**; the server keeps device capabilities in its `device` table (spec 07). Richer capability and driver-identity reporting is open ([[Open-Questions]] Q34).
- **Always available:** the `QUERY STATUS` byte, actual level and bank-0 identity. That preserves per-light **fault identity**, the most valuable piece.
- **Fail-safe levels are set by command**, not procurement, so they work on any DALI generation.
- Treat 251/252/253 as **optional enrichment**. Nothing in control, fault detection or fail-safe may depend on them.

## Who does what

| Step | Done by |
|---|---|
| DALI queries, memory-bank reads, caching on the device | Per-pole device (device developer) |
| Reporting capabilities and telemetry over LoRa | Device-role protocol library + device firmware |
| Forwarding the `d4i_available` bit and "not available" values, relaying `REPLACE_DRIVER` | Slave/Master gateways |
| Showing capabilities, not alarming on absent data | Local Monitoring Server |

## D4i in brief

D4i = DALI-2 (101 + 102 + 207) + Part 250 (integrated bus supply) + 251 + 252 + 253 + 351. If the existing drivers happen to be D4i, they also bring:

- **Integrated DALI bus PSU** in the driver, so the per-pole device need not supply bus power.
- **Optional auxiliary supply (Part 150): 24 V DC, 125 mA, 3 W average / 6 W peak.** Enough to power the per-pole device (STM32F103 + SX1276) from the driver. ⚠ Part 150 aux is not required for D4i certification; check per model.
- Alignment with **ANSI C136.41** (7-pin twist-lock carries DALI + low-voltage supply) and **Zhaga Book 18** (4-pin, DALI + aux) sockets.

## Where the data lives

Read via `READ MEMORY LOCATION`, **one byte per ~25 ms** ([[DALI-Fundamentals]]):

| Bank | Part | Content |
|---|---|---|
| 0 | 102 | GTIN, firmware/hardware version, serial number, DALI versions |
| 1 | 251 | **Luminaire asset data** |
| **202** | 252 | **Active energy + active power** (mandatory under D4i) |
| 203/204 | 252 | Apparent energy/power; load-side energy/power (optional) |
| **205** | 253 | **Gear diagnostics and maintenance** |
| **206** | 253 | **Light-source diagnostics and maintenance** |
| 207 | 253 | Luminaire maintenance data |

**Part 251 (asset data, written by the luminaire OEM):** luminaire GTIN and ID, date code, **nominal input power**, power at minimum dim, nominal mains voltage range, **nominal light output (lm)**, CRI, **CCT**, distribution type, 60-character description. `nominal_input_power` drives expected-vs-measured power reconciliation ([[Energy-Metering-And-Telemetry-Contract]]).

**Part 252 (energy):** active energy (Wh) and power (W). **Raw integers plus a scaling factor; read and apply the scaler.**

⚠ **Part 252 energy is not revenue-grade.** Accuracy is set by the driver maker (one quotes ~1%), and billing on it is restricted in many jurisdictions. Use it to verify savings and check luminaire health; revenue metering is the section meter's job.

**Part 253 (diagnostics):**

1. **Performance data:** supply voltage and frequency, **power factor**, **driver temperature**, output current %, light-source voltage/current/temperature.
2. **Failure flags.** Driver: overall, under-voltage, over-voltage, output power limitation, thermal derating, thermal shutdown. Light source: overall, **open circuit**, **short circuit**, thermal derating, thermal shutdown.
3. **Failure flag counters**: 0–253 per flag, non-volatile, no timestamps.
4. **Timers** (4 bytes, 1 s resolution): driver operating time, light-source on-time (lifetime and resettable).
5. **Lifetime counters** (up to 16 million): driver starts, light-source starts.

Plus OEM operation info: rated median useful life, internal reference temperature, rated starts.

⚠ **Part 253 thresholds are not standardised.** "Thermal derating" from vendor A and vendor B are not comparable, so **store the driver GTIN/model with every flag.** Flags reset on power cycle, so **poll counters, not just flags.** Frequent Philippine outages will push the driver start counter up fast; that is not a fault ([[Philippines-Site-Context]]).

## Capability discovery

Run per driver at commissioning and after any driver swap:

| Step | Query | Tells you |
|---|---|---|
| 1 | `QUERY DEVICE TYPE` (0x99) | 6 = LED driver; anything else voids assumptions |
| 2 | `QUERY VERSION NUMBER` (0x97) | Part 102 version: DALI-1 vs DALI-2 |
| 3 | Read **bank 0** | GTIN, firmware/hardware version, serial, declared DALI versions |
| 4 | Probe **bank 202** | Part 252 energy present |
| 5 | Probe **banks 205/206** | Part 253 diagnostics present |
| 6 | Probe **bank 1** | Part 251 asset data, incl. `nominal_input_power` |
| 7 | `QUERY PHYSICAL MINIMUM` (0x9A) | Real dimming floor for this driver |
| 8 | Bank 0 / Part 250 behaviour | Integrated bus PSU present |

Then:

- The Modbus map keeps the **full fixed-stride per-device block**. A field the device can't measure is sent as "not available" (all bits set, e.g. `0xFFFF`), never as zero (spec 00 §2.2, spec 01).
- The device sets `d4i_available` in its status bits, and uses Part 252 for its power and energy fields where the driver has it.
- **Probe once, cache forever** for a given physical driver.

**First real driver, on the bench (2026-10-28):** the LTECH SE-12-350-700-W1DS answered steps 1–3 as DT6, DALI-2 (version 0x08), with bank 0 giving firmware 2.0 and hardware 1.0 but **a blank GTIN** (all 0xFF), so identify drivers by the bank-0 identification number. Bank 1 exists; banks 202, 205 and 206 are not yet probed. It has no bus supply. See [[DALI-Bench-Rig-And-Results]].

## If the drivers are plain DALI-2 or DALI-1

| Lost | Consequence |
|---|---|
| Part 252 energy | No per-light energy from the driver; section energy comes from the meter (totals, not attribution) |
| Part 253 diagnostics | No predictive maintenance; faults become binary and reactive |
| Part 251 asset data | No `nominal_input_power`; keep a hand-entered per-model constant in the registry |
| Part 150 aux supply | The per-pole device needs its own power supply |
| Part 250 integrated PSU | The per-pole device must supply DALI bus power |

**Still available:** the status byte (lamp failure, gear failure, limit error, missing address, power cycle) and actual level. Per-light fault identity and meter reconciliation survive; only the *why* is lost.

## Configure regardless of driver generation

Mandatory in Part 102 and set by command:

- **`SYSTEM FAILURE LEVEL` = 254 (100%)**, **`POWER ON LEVEL` = 254 (100%)**. Client option: System Failure = 250 (≈90%) to tell the cases apart on the road (spec 05 §6, [[Open-Questions]] Q31). Fail-safe layer 4 in [[Fail-Safe-Lighting-Rule]].
- **`MIN` / `MAX LEVEL`**, above the queried physical minimum. With mixed models, set a common MIN at or above the highest physical minimum to avoid uneven low-end dimming.
- **`FADE TIME`**: index 7–10 (5.7–16 s) for scheduled changes (spec 05 §6). Longer fades are possible but slow down group switching.

⚠ **None of this survives a driver swap.** See [[Commissioning-And-Field-Swap]].

## Per-light telemetry contract, tiered

Cached with its age ([[Modbus-Register-Map-Design]]):

- **Always:** status (mapped to the device status bits in spec 03 §6), actual and target level. ⚠ Driver GTIN/serial has no field in v0.1 (Q34).
- **If Part 252:** active power and cumulative energy, scaler applied.
- **If Part 253:** driver temperature and thermal-derating flag (real events expected on west-facing poles in April–May), on-time and start counters against rated figures, failure-flag counters.

## Alternatives to DALI, for context

| Interface | Verdict |
|---|---|
| **0–10 V / 1–10 V** | No feedback, no addressing; ~90 m before uneven dimming; noise-prone; 1–10 V has no off state |
| PWM | Noise-immune but one-way and unaddressed |
| **DALI / D4i** | Addressable, bidirectional, real telemetry |
| Mains switching | On/off only |

US DOE/PNNL **PNNL-32949** tested 19 LED streetlight drivers: **9 of 19 non-compliant** with ANSI C137.1-2022, with power across the control range varying by **~47 percentage points** between units. See [[Street-Lighting-Control-Logic]].

## Related

- [[DALI-Fundamentals]]
- [[Energy-Metering-And-Telemetry-Contract]]
- [[Commissioning-And-Field-Swap]]
- [[Device-Identity-Namespaces]]
- [[DALI-Bench-Rig-And-Results]]

## Sources

- `Specs/specs/03-tier2-device-protocol.md` §6–§7, `Specs/specs/05-fail-safe-behaviour.md` §6 (v0.1).
- Research synthesis, 2026-09-09. DALI Data: https://www.dali-alliance.org/dali/data.html · D4i: https://www.dali-alliance.org/d4i/ · Inventronics D4i technical primer: https://www.inventronicsglobal.com/am/wp-content/uploads/2020/07/DALI-DALI-2-and-D4i-Technical-Primer.pdf · Beckhoff Part 252/253 bank docs: https://infosys.beckhoff.com/content/1033/tcplclib_tc3_dali/8713097611.html · PNNL-32949: https://www.pnnl.gov/main/publications/external/technical_reports/PNNL-32949.pdf

## History

- 2026-09-09: recommended specifying D4i drivers as a procurement lever.
- 2026-09-13: existing off-the-shelf drivers chosen; note became a discovery and graceful-degradation design.
- 2026-10-14: per-pole devices; discovery now runs on the device, with the gateways storing the result.
- 2026-10-21: aligned with spec pack v0.1 (only a d4i_available bit on the radio; not-available sentinels instead of quality words; fade 7–10; fail-safe levels 254 with 250 option; driver identity open as Q34).
- 2026-10-28: first real driver probed on the bench: DT6, DALI-2, blank GTIN, no bus supply ([[DALI-Bench-Rig-And-Results]]).

Back to [[Index]]
