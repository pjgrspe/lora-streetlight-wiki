# DALI Telemetry And D4i

What per-light data the gateway can actually get — and, since **the project is using existing off-the-shelf DALI drivers rather than specifying or developing them** (decision, 2026-09-13), how to discover what's actually available rather than assume it.

## This is now a verification task, not a procurement lever

For **DALI-2** certification, Parts 251/252/253 are all **optional**. For **D4i**, all three are **mandatory** (device types 50/51/52).

> **Originally this note recommended specifying D4i drivers as the highest-leverage procurement decision in the project. That lever is no longer available.** With existing drivers, the question changes from *"what do we require?"* to *"what did we get?"* — and the answer may be nothing beyond the basic status byte.
>
> **Consequence: the telemetry contract must be discovered per driver model and degrade gracefully.** See *Capability discovery* below.

D4i = DALI-2 (101 + 102 + 207) + Part 250 (integrated bus supply) + 251 + 252 + 253 + 351. If the existing drivers happen to be D4i-certified, it also brings:

- **Integrated DALI bus PSU in the driver** — no separate bus PSU in the pole. (But see the 250 mA collision in [[DALI-Bus-Length-Constraint]].)
- **Optional auxiliary supply (Part 150): 24 V DC, 125 mA, 3 W average / 6 W peak** — enough to power a LoRa radio module directly from the driver. **A big deal: no AC power pack, no separate PSU in the pole-top node.** ⚠ Part 150 aux is *not* required for D4i certification, so specify it explicitly if you want it.
- Alignment with **ANSI C136.41** (3/5/7-pin twist-lock roadway receptacle; the 7-pin carries a DALI pair plus low-voltage supply) and **Zhaga Book 18** (4-pin, DALI + aux). Either socket is what lets a city **field-upgrade the radio without touching the luminaire**.

## Where the data lives

Standardised memory banks, read via `READ MEMORY LOCATION` — **one byte per ~25 ms round trip**, which is why deep telemetry must be scheduled and cached ([[DALI-Fundamentals]]).

| Bank | Part | Content |
|---|---|---|
| 0 | 102 | GTIN, firmware/hardware version, serial number, DALI versions |
| 1 | 251 | **Luminaire asset data** |
| **202** | 252 | **Active energy + active power — MANDATORY under D4i** |
| 203/204 | 252 | Apparent energy/power; load-side energy/power (optional) |
| **205** | 253 | **Gear diagnostics and maintenance** |
| **206** | 253 | **Light-source diagnostics and maintenance** |
| 207 | 253 | Luminaire maintenance data |

**Part 251 (asset data, written by the luminaire OEM):** luminaire GTIN and identification number, manufacturer date code, **nominal input power**, power at minimum dimming level, nominal min/max mains voltage, **nominal light output (lm)**, CRI, **CCT**, light distribution type, and a 60-character description field. `nominal_input_power` is what makes expected-vs-measured power reconciliation possible — see [[Energy-Metering-And-Telemetry-Contract]].

**Part 252 (energy):** active energy (Wh) and active power (W) mandatory. **Values are raw integers with an accompanying scaling factor — you must read and apply the scaler, not assume units.**

> **⚠ Do not treat Part 252 energy as revenue-grade.** Per Part 252, *the accuracy of these measurements is defined by the driver manufacturer* (one vendor quotes ~1%), and in many jurisdictions law restricts using such a device for billing. The vendor's own recommended use is *"to track and verify energy savings, and check the health of the luminaire."* Revenue-grade metering is the separate 3-phase meter's job.

**Part 253 (diagnostics)** — five mechanisms, and this is the predictive-maintenance layer:

1. **Performance data:** driver supply voltage and frequency, **driver power factor**, **driver temperature**, output current percentage, light-source voltage/current/temperature.
2. **Failure flags** — driver: overall, supply under-voltage, supply over-voltage, output power limitation, thermal derating, thermal shutdown. Light source: overall, **open circuit**, **short circuit**, thermal derating, thermal shutdown.
3. **Failure flag counters** — one per flag, 0–253 occurrences, non-volatile, read-only. No timestamps, just counts.
4. **Timers** — 4 bytes each at 1 s resolution (>136 years of range): driver operating time, light-source on-time (lifetime, non-resettable), light-source on-time (resettable).
5. **Lifetime counters** — up to 16 million: driver start counter, light-source start counter.

Plus OEM-populated luminaire operation info: rated median useful life, internal reference temperature, rated median useful light starts.

## Two caveats that shape the data model

> **⚠ Part 253 thresholds are NOT standardised.** Each failure-flag threshold is set by the driver or luminaire manufacturer. So "thermal derating = true" from vendor A and vendor B **are not comparable.**
>
> **Therefore: store the driver GTIN/model alongside every flag** so the server can interpret it.

**Flags reset on power cycle or after hysteresis, so poll the counters, not just the flags,** if you want a true fault history. In the Philippines, where outages are frequent, expect the **driver start counter to climb fast — don't misread that as a fault** ([[Philippines-Site-Context]]).

## Capability discovery — a required gateway feature

Because the drivers are pre-existing, **the gateway cannot assume a telemetry set. It must probe for one at commissioning and record what it found in the device registry.**

Discovery routine, run per driver during commissioning and re-run after any driver swap:

| Step | Query | Tells you |
|---|---|---|
| 1 | `QUERY DEVICE TYPE` (0x99) | 6 = LED driver. Anything else and your assumptions are void |
| 2 | `QUERY VERSION NUMBER` (0x97) | Part 102 version — DALI-1 vs DALI-2 generation |
| 3 | Read **memory bank 0** | GTIN, firmware/hardware version, serial number, and the declared DALI version for Parts 101/102/103 |
| 4 | Probe **bank 202** | Present → Part 252 energy reporting exists |
| 5 | Probe **banks 205/206** | Present → Part 253 diagnostics exist |
| 6 | Probe **bank 1** | Present → Part 251 luminaire asset data, incl. `nominal_input_power` |
| 7 | `QUERY PHYSICAL MINIMUM` (0x9A) | The real dimming floor for *this* driver |
| 8 | Read bank 0 for Part 250 behaviour | Whether the driver has an integrated bus PSU that must be disabled |

**Store the result as a capability bitmap in the registry, keyed by driver GTIN**, alongside the fields listed in [[Device-Identity-Namespaces]]. Then:

- The Modbus register map exposes the **full** per-luminaire block always (fixed stride — don't reshape it per driver), but unsupported fields are marked `NEVER_READ` in their quality word rather than reporting zero. See [[Cache-Behind-Synchronous-Wire-Pattern]].
- A per-section `telemetry_capability` word tells the server what to expect, so it doesn't alarm on permanently-absent data.
- **Probe once, cache forever.** Discovery is expensive (one byte per ~25 ms round trip) and the answer never changes for a given physical driver.

## If the drivers turn out to be plain DALI-2 or DALI-1

The realistic downside case, and it must be designed for rather than discovered late:

| Lost | Consequence |
|---|---|
| Part 252 energy | **No per-light energy.** Section energy comes only from the 3-phase meter — you get totals, not attribution |
| Part 253 diagnostics | **No predictive maintenance.** No driver temperature, no thermal-derating counters, no open/short discrimination, no operating-hours trending. Faults become binary and reactive |
| Part 251 asset data | **No `nominal_input_power`**, which is the input to expected-vs-measured power reconciliation — so that fault-detection layer degrades to a hand-maintained per-model constant in the registry |
| Part 150 aux supply | **No 24 V for the radio node** — a pole-top node needs its own power pack |
| Part 250 integrated PSU | Need a **separate DALI bus power supply** per bus, sized and installed |

**What still works, and it is not nothing:** the `QUERY STATUS` byte is mandatory in every DALI generation. Lamp failure, gear failure, limit error, missing short address, power-cycle-since-last-query and actual level are all still available. **That preserves per-light fault *identity*** — which is the single most valuable piece of telemetry — and it keeps failure-detection layers 1 and 3 intact in [[Energy-Metering-And-Telemetry-Contract]]. Only layer 2 (the *why*) is lost.

**Design rule:** treat Parts 251/252/253 as optional enrichment throughout. Nothing in the gateway's core function — control, fault detection, fail-safe — may depend on them.

## What still needs configuring, existing drivers or not

These are set by *command*, not by procurement, so they remain fully in your control — and they are mandatory in Part 102 regardless of DALI generation:

- **`SYSTEM FAILURE LEVEL` = 100%** and **`POWER ON LEVEL` = 100%** (or 90%/100% to make the two distinguishable on the road). This is fail-safe layer 4 in [[Fail-Safe-Lighting-Rule]], and it works even on the oldest DALI-1 driver.
- **`MIN` / `MAX LEVEL`**, clamped above the driver's queried physical minimum.
- **`FADE TIME`** — long fades (16–90 s) for roadway use.
- Group and scene memberships.

⚠ **None of this survives a driver swap** — a replacement arrives in factory state. See the re-addressing procedure in [[Commissioning-And-Field-Swap]].

⚠ **Mixed driver models are now likely.** Existing stock is rarely homogeneous, and drivers with different physical minima in one visual field produce **visibly uneven dimming at the bottom of the range**. Query the physical minimum per driver and set a common `MIN LEVEL` at or above the highest of them.

## What to ask for at the gateway boundary

Per luminaire, cached with age and quality ([[Modbus-Register-Map-Design]]). **Tiered by what discovery actually found:**

**Always available (any DALI generation) — the core contract:**

- Status byte (lamp fail, gear fail, limit error, missing address, power cycle)
- Actual level and commanded level
- Driver GTIN/serial from bank 0, so a swap is auditable

**Only if Part 252 is present:**

- Active power and cumulative active energy, with the scaler applied

**Only if Part 253 is present:**

- Driver temperature and thermal-derating flag — **genuinely useful in a tropical install, not decorative**; expect real derating events on west-facing poles in April–May
- Light-source on-time and start counter, versus the rated figures from Part 251
- Failure flag *counters*, not just current flags (flags reset on power cycle, and PH outages are frequent)

## Alternatives to DALI, for context

| Interface | Verdict for street lighting |
|---|---|
| **0–10 V / 1–10 V** | **No feedback whatsoever** — no telemetry, no addressing, no fault reporting. ~90 m distance limit before uneven dimming; susceptible to induced noise and ground-potential differences. 1–10 V additionally **has no off state**. |
| PWM | Noise-immune, good low-end stability, but still one-way and unaddressed |
| **DALI / D4i** | Addressable, bidirectional, real telemetry |
| Mains switching only | On/off, no dimming |

**Hard evidence against 0–10 V:** the US DOE/PNNL study **PNNL-32949** measured 19 LED streetlight drivers against ANSI C137.1-2022 and found **nine of 19 non-compliant**, with relative power draw across the control range **varying by ~47 percentage points between units** — i.e. "5 V" means wildly different power on different luminaires. See the measured savings shortfall in [[Street-Lighting-Control-Logic]]. This is the strongest available argument for specifying D4i.

## Related

- [[DALI-Fundamentals]]
- [[Energy-Metering-And-Telemetry-Contract]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. DALI Data: https://www.dali-alliance.org/dali/data.html · D4i: https://www.dali-alliance.org/d4i/ · Inventronics D4i technical primer: https://www.inventronicsglobal.com/am/wp-content/uploads/2020/07/DALI-DALI-2-and-D4i-Technical-Primer.pdf · Beckhoff Part 252/253 bank docs: https://infosys.beckhoff.com/content/1033/tcplclib_tc3_dali/8713097611.html · PNNL-32949: https://www.pnnl.gov/main/publications/external/technical_reports/PNNL-32949.pdf
