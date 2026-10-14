# Energy Metering And The Telemetry Contract

What the 3 section meters measure, what the monitoring server actually wants, and how failure detection is layered.

## What a 3-phase meter provides

V per phase (and line-line), I per phase plus neutral, active/reactive/apparent power, power factor, frequency, THD-V and THD-I, cumulative kWh import/export, kvarh, max demand.

**Accuracy classes:** IEC 62053-21 (direct connect) → classes 0.5/1/2 in Ed.2:2020. IEC 62053-22 (transformer-operated) → 0.2S/0.5S, with 0.1S added in Ed.2. ⚠ IEC 62053-23 (reactive) **is being withdrawn** — its classes folded into 62053-24; do not specify it for a new design.

The **"S" suffix** means designed for instrument-transformer connection with tightened accuracy **down to 1% of nominal current** — which is exactly the regime a street-lighting circuit sits in during deep dimming.

**Recommendation: Class 1 (IEC 62053-21) is sufficient.** These are the customer's own instrumentation, not revenue meters (see [[Philippines-Site-Context]]), so no utility sealing or approval is needed. ⚠ **But check accuracy at low load:** at 20% dim across 128 lights the load drops to ~2.6 kW, so **size CTs so normal operating current sits in the 20–100% band, not 2%.**

**Direct-connect vs CT:** at ~13 kW / 230 V 3-phase (≈19 A/phase), a direct-connect meter is entirely adequate and **eliminates CT ratio, polarity and burden errors — the most common metering installation mistake.** If CTs are used, **expose the configured CT and PT ratios as readable attributes** so the server can detect a mis-scaled meter ("energy is 5x wrong" is almost always a CT ratio).

## Modbus register conventions

The **Eastron SDM630** layout is the widely-cloned reference:

- **Function code 04 (Read Input Registers)** for all measurements. **FC03 on a measurement address returns an exception — a very common integration bug.**
- **Every measurement is a 32-bit IEEE-754 float across two consecutive registers, high word first.** No integer-plus-scale-factor in the measurement area.
- FC03/FC16 apply to holding registers for configuration (baud, address, CT ratio).
- Serial default commonly 9600 8N1.

| Register (1-based, FC04) | Parameter |
|---|---|
| 30001/30003/30005 | Voltage L1-N / L2-N / L3-N |
| 30007/30009/30011 | Current L1/L2/L3 |
| 30053 | **Total active power** |
| 30071 | Total power factor |
| 30073/30075 | **Total import / export active energy** |
| 30077 | Frequency |

⚠ **Off-by-one warning:** vendors document these as either 1-based register numbers (30001) or 0-based protocol addresses (0x0000). **Confirm against a live meter before freezing your map.**

## The telemetry contract

**Per section, from the 3 meters:**

| Priority | Point | Cadence | Why |
|---|---|---|---|
| 1 | **ΣkWh import** (cumulative, monotonic) | 5 min | Ground truth for savings verification and bill reconciliation |
| 2 | P per phase and total | 60 s | The primary fault-detection signal |
| 3 | V per phase | 60 s | Brownout/over-voltage detection; PH LV excursion is significant, and **V is a leading indicator of driver failures** |
| 4 | I per phase + **I_neutral** | 60 s | Unbalance detection |
| 5 | PF total | 60 s | Driver-population health |
| 6 | Frequency | 60 s | Grid-event correlation |
| 7 | THD-I | daily | Driver degradation trend |

**Derived at the server:**

- **Unbalance %** = `(max|I_ph − I_avg|) / I_avg × 100`.

  > **⚠ The phase allocation in the diagram is significantly unbalanced by design:** 53/38/37 against an average of 42.7 means **+24% on phase 1**. Expect permanently elevated neutral current, and **set unbalance alarm thresholds against the designed baseline, not against 0%.** Rebalance at survey time if the wiring allows — see [[Commissioning-And-Field-Swap]].

- **Expected power** = Σ `nominal_input_power` (Part 251) × `dimcurve(commanded_level)`. Compare to measured.
- **Lights-out estimate** = `(P_expected − P_measured) / P_per_light`. With ~100 W luminaires and a Class-1 meter this resolves a *single* failed light out of 53 (100 W of 5.3 kW ≈ 1.9%, above 1% meter error) — **but only just, and only if commanded levels are known exactly.**
- **Day-burner detection:** P > 0 during the astronomically computed daytime window.

## Failure detection is three independent layers

| Layer | Source | Gives you | Available with existing drivers? |
|---|---|---|---|
| 1 | **DALI status byte per light** | Authoritative *identity* of the failed light — requires per-light addressing | **Yes — mandatory in every DALI generation** |
| 2 | **DALI Part 253 flags and counters** | *Why* it failed (thermal, over-voltage, open circuit), and degradation before failure | **⚠ Only if the drivers are D4i** — see [[DALI-Telemetry-And-D4i]] |
| 3 | **Meter power reconciliation** | Lights out (step down in P), day burners (P>0 in daytime), cycling drivers (oscillating P), feeder faults (whole phase drops), theft/unauthorised load (P above expected) | Yes |

> **Layer 3 alone gives counts, not identities. Layers 1 and 2 give identities — but only if DALI addressing survives the topology decision in [[DALI-Bus-Length-Constraint]]. That is the strongest argument for per-pole nodes.**

**⚠ Since the project is using existing off-the-shelf drivers (2026-09-13), layer 2 may simply not exist.** Layers 1 and 3 survive regardless, so fault *detection* and *identity* are safe — you lose the *why*, and therefore predictive maintenance. Design the alarm model so layer 2 is enrichment, never a dependency.

**One knock-on for layer 3:** expected-power reconciliation needs `nominal_input_power`, which comes from DALI Part 251. If the drivers don't expose bank 1, **keep a per-driver-model nominal-power constant in the registry**, populated by hand at commissioning from the luminaire datasheet. Slightly worse (it won't track a swapped-in different model automatically), but it keeps the layer working.

All three must land in the gateway cache with **age and quality**, never as bare values — see [[Cache-Behind-Synchronous-Wire-Pattern]].

## Related

- [[DALI-Telemetry-And-D4i]]
- [[Modbus-Register-Map-Design]]
- [[Failure-Modes-And-Mitigations]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. IEC 62053-22:2020: https://webstore.iec.ch/en/publication/29987 · Eastron SDM630 register list: https://modbusmanager.com/eastron-sdm630-modbus.html · fault signatures: https://www.frontiersin.org/journals/sensors/articles/10.3389/fsens.2026.1905827/full
