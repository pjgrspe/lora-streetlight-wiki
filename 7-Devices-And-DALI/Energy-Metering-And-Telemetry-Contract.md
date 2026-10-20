# Energy Metering And The Telemetry Contract

What the section energy meters measure, what the server needs from them, and how fault detection is layered across meters and per-pole devices.

## Key points

- **One LoRa energy meter per section** (device developer), reporting through that section's Slave Gateway.
- **Class 1 (IEC 62053-21)** is enough: this is the customer's own instrumentation, not revenue metering. Prefer **direct-connect** over CTs.
- **Fault detection has three independent layers:** per-pole status (identity), driver diagnostics (the *why*, only if D4i), and meter power reconciliation (counts).
- With per-pole devices, **per-light fault identity is guaranteed**; only the diagnostic *why* depends on driver capability.
- Every value lands in the gateway cache with **its age**, never bare ([[Cache-Behind-Synchronous-Wire-Pattern]]).
- **On the radio:** the meter sends a 44-byte REPORT in a double slot every cycle (120 s) to its slave; the slave forwards a meter record to the master every 5 min. Per-pole devices send a 20-byte REPORT every cycle. Specified in `Specs/specs/03-tier2-device-protocol.md` §7 (v0.1).

## What a 3-phase meter provides

V per phase (and line-line), I per phase plus neutral, active/reactive/apparent power, power factor, frequency, THD-V and THD-I, cumulative kWh import/export, kvarh, max demand.

**Accuracy classes:** IEC 62053-21 (direct connect): classes 0.5/1/2 (Ed.2:2020). IEC 62053-22 (transformer-operated): 0.2S/0.5S, plus 0.1S in Ed.2. ⚠ IEC 62053-23 (reactive) is being withdrawn into 62053-24; don't specify it.

The **"S" suffix** means tightened accuracy **down to 1% of nominal current**, which is where a lighting circuit sits during deep dimming.

⚠ **Check low-load accuracy:** at 20% dim across the diagrammed 128 lights, load drops to ~2.6 kW. If CTs are used, size them so normal current sits in the 20–100% band.

**Direct-connect vs CT:** at ~13 kW / 230 V three-phase (≈19 A/phase) for 128 lights, direct-connect is adequate and **removes CT ratio, polarity and burden errors**, the most common metering mistake. If CTs are used, expose the CT/PT ratios as readable attributes ("energy is 5× wrong" is almost always a CT ratio).

## Modbus register conventions

The meter is behind a LoRa device, but its internal register conventions still matter to whoever reads it. The **Eastron SDM630** layout is the widely cloned reference:

- **FC04 (Read Input Registers)** for all measurements. **FC03 on a measurement address returns an exception**, a common bug.
- **Each measurement is a 32-bit IEEE-754 float in two registers, high word first.**
- FC03/FC16 for configuration (baud, address, CT ratio). Default often 9600 8N1.

| Register (1-based, FC04) | Parameter |
|---|---|
| 30001/30003/30005 | Voltage L1-N / L2-N / L3-N |
| 30007/30009/30011 | Current L1/L2/L3 |
| 30053 | **Total active power** |
| 30071 | Total power factor |
| 30073/30075 | **Total import / export active energy** |
| 30077 | Frequency |

⚠ **Off-by-one:** vendors document 1-based numbers (30001) or 0-based addresses (0x0000). Confirm against a live meter.

## Telemetry contract per section meter

| Priority | Point | Why |
|---|---|---|
| 1 | **ΣkWh import** (cumulative, monotonic), plus export | Ground truth for savings and bill reconciliation |
| 2 | P per phase and total | Primary fault-detection signal |
| 3 | V per phase | Brownout/over-voltage; **a leading indicator of driver failures** |
| 4 | I per phase + **I_neutral** | Unbalance |
| 5 | PF total | Driver-population health |
| 6 | Frequency | Grid-event correlation |
| 7 | THD-I | Driver degradation trend |

**All seven are in every meter REPORT** (44 bytes, plus wye/delta and phase-loss flags and a fault code), sent once per cycle. The cadence at each hop:

| Hop | Cadence |
|---|---|
| Meter → slave | Every cycle (120 s default) |
| Slave → master | Meter record every 5 min; phase loss and mains loss as immediate events |
| Master → server | Whenever the server polls the meter block (spec 01) |

The 5-minute forwarding interval comes from the slave's duty-cycle budget ([[LoRa-Airtime-Budget]]). Phase loss doesn't wait for it: the meter raises `METER_PHASE_LOSS` and the slave raises `PHASE_LOSS` / `MAINS_LOSS` events at once.

## Per-pole device telemetry

Each LED device's 20-byte REPORT carries: status bits, actual and target level (DALI arc 0–254), input power, supply voltage and current, cumulative energy, temperature, lamp hours, a fault code and the last executed command. Fields the hardware can't measure are sent as "not available" (all bits set). The slave forwards basic per-device records (status, level, power, energy) every 15 min, and extended ones (voltage, current, lamp hours, link quality) hourly. Status changes go up at once.

## Derived at the server

- **Unbalance %** = `(max|I_ph − I_avg|) / I_avg × 100`. ⚠ The diagrammed allocation (53/38/37, average 42.7) is **+24% on phase 1** by design. Set unbalance alarms against the **designed baseline**, not 0%, and rebalance at survey time if wiring allows.
- **Expected power** = Σ `nominal_input_power` (Part 251, or a per-model constant) × `dimcurve(commanded_level)`.
- **Lights-out estimate** = `(P_expected − P_measured) / P_per_light`. With ~100 W lights and a Class-1 meter this resolves one failed light in 53 (1.9% vs 1% error), **only just, and only if commanded levels are exact**.
- **Day-burner detection:** P > 0 during the astronomically computed daytime.

## Three independent fault-detection layers

| Layer | Source | Gives you | Status |
|---|---|---|---|
| 1 | **Per-pole device: DALI status byte** | Authoritative *identity* of the failed light | **Always available**: per-pole addressing, mandatory in every DALI generation |
| 2 | **DALI Part 253 flags and counters** | *Why* it failed, and degradation before failure | ⚠ **Only if the drivers are D4i** ([[DALI-Telemetry-And-D4i]]) |
| 3 | **Meter power reconciliation** | Lights out, day burners, cycling drivers, feeder faults, theft/unauthorised load | Always available |

Layer 3 alone gives counts, not identities; layer 1 now provides identity because every pole has its own device. Design alarms so **layer 2 is enrichment, never a dependency**. If Part 251 is missing, keep a per-model `nominal_input_power` constant in the registry, entered at commissioning from the datasheet.

## Related

- [[DALI-Telemetry-And-D4i]]
- [[Modbus-Register-Map-Design]]
- [[Failure-Modes-And-Mitigations]]
- [[Street-Lighting-Control-Logic]]

## Sources

- `Specs/specs/03-tier2-device-protocol.md` §7, `Specs/specs/02-tier1-protocol.md` (DETAIL records), `Specs/specs/04-channel-plan-and-airtime.md` §7 (v0.1).
- Research synthesis, 2026-09-09. IEC 62053-22:2020: https://webstore.iec.ch/en/publication/29987 · Eastron SDM630 register list: https://modbusmanager.com/eastron-sdm630-modbus.html · fault signatures: https://www.frontiersin.org/journals/sensors/articles/10.3389/fsens.2026.1905827/full

## History

- 2026-09-13: existing off-the-shelf drivers chosen; layer 2 became optional.
- 2026-10-14: per-pole devices make layer 1 (identity) always available; meters are one per section with the section count not fixed.
- 2026-10-21: aligned with spec pack v0.1 (meter and LED REPORT contents, per-hop cadence: every 120 s cycle to the slave, 5 min meter / 15 min device detail to the master).

Back to [[Index]]
