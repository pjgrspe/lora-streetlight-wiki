# Street Lighting Control Logic

How the lights decide when to switch and how far to dim. The decision: **astronomical clock as primary, computed locally in every gateway and device**, with photocells only as an override-on input, and energy **measured rather than modelled**.

## Key points

- **Astronomical schedule computed locally** from lat/long + date, in the Master, every Slave Gateway, and (as a fallback) every per-pole device.
- **The nightly on/off costs zero radio traffic.** The network is for supervision and override, not the routine switch.
- Photocells may only force lights **on**, never off.
- **Measured savings run well below modelled ones** (PNNL): compare commanded → predicted → measured power.
- ⚠ Dimming depth is legally bounded by road-class lux, and **dimming saves nothing on a flat (unmetered) service**.

## Photocell vs astronomical clock

| | Photocell | Astronomical clock |
|---|---|---|
| Trigger | Measured ambient lux | Computed sunrise/sunset |
| Failure mode | **Day-burner** (dirty/failed cell) or dark street | Clock drift, wrong coordinates or timezone |
| Storms | **Handles them**: typhoon rain or ash can make lights legitimately needed | Stays off through a dark afternoon |
| Consistency | Poor: each cell switches at a slightly different time | Excellent: one computed schedule |
| Energy predictability | Poor | Good |

**Astronomical primary, photocell override-on only.** The algorithm and Manila's figures are in [[Time-Distribution-And-Clock-Trust]].

## Where the schedule lives

| Level | Role |
|---|---|
| Local Monitoring Server | Edits offsets and dimming profiles; pushes them down. Not in the run-time path |
| Master Gateway | Computes and distributes; validates |
| Slave Gateways | **Decide autonomously** for their section ([[Slave-Gateways-As-Intelligent-Sub-Controllers]]) |
| Per-pole devices | Hold a **fallback schedule** so a dead slave doesn't leave the pole uncontrolled (device developer) |
| DALI driver | `SYSTEM FAILURE LEVEL` / `POWER ON LEVEL` = 100% as the last layer ([[Fail-Safe-Lighting-Rule]]) |

## Dimming profiles

1. **Full-night constant**: baseline.
2. **Midnight / part-night dimming**: e.g. 100% to 00:00, 50% to 05:00, 100% to dawn.
3. **Multi-step adaptive**: 100/70/50/70% by traffic band. Sent as group commands with `execute_at`, so a whole section changes together ([[Popcorn-Effect-And-Scheduled-Execution]]).
4. **Part-night switch-off**: **not on public roads** without an explicit LGU safety and liability decision.
5. **CLO (Constant Light Output)**: start dimmed and raise power as the LED depreciates (L80@16 yr ≈ start 20% dimmed, add 20 points linearly). ⚠ CLO can live in the driver **or** in the control system (using Part 253 on-time). **Pick one**; both double-compensate.

## ⚠ Measured savings fall short

PNNL-32949:

| Strategy | Expected | **Measured** |
|---|---|---|
| Part-night dimming | 18% | **12%** |
| CLO L90@20yr | 5% | **≈0%** |
| CLO L80@16yr | 10% | **−2%** |

PNNL's advice: calibrate to each luminaire's measured dimming curve; don't assume linearity or compliance. **For this system: the server compares commanded level → predicted power → measured power** and treats divergence as a fault signal and a calibration input ([[Energy-Metering-And-Telemetry-Contract]]).

## Energy math (diagrammed 128 lights)

128 luminaires × 100 W input, 12.0 h average night:

| Scenario | Nightly kWh | Annual kWh | @ ₱11/kWh |
|---|---|---|---|
| HPS baseline (250 W HPS ≈ 275 W input) | 422 | 154,200 | ₱1,696,000 |
| **LED 100%, all night** | 153.6 | 56,100 | ₱617,000 |
| LED + midnight dim | 115.2 | 42,100 | ₱463,000 |
| LED + 4-step adaptive (~35% reduction) | 99.8 | 36,400 | ₱401,000 |

LED retrofit alone saves **~64%** versus HPS. Dimming adds a nominal 25%; **expect ~17% realised** after the PNNL de-rating if dimming isn't calibrated. Scale linearly for the actual light count.

⚠ **On an unmetered "flat streetlight" service, dimming saves zero pesos.** Answer this before investing in dimming logic ([[Philippines-Site-Context]]).

## Legal floor on dimming depth

⚠ The **DOE Roadway Lighting Guidelines (2017)** and **DPWH DO No. 19** set illuminance by road class (secondary sources: Class C feeder roads ≥10–15 lux, uniformity ≥0.3; CCT 2,000–6,500 K, warm white on arterials). The primary DOE PDF could not be retrieved. **MIN LEVEL and dimming profiles must not go below the mandated lux and uniformity.** Get the official documents before fixing dim floors ([[Open-Questions]]).

## Related

- [[Time-Distribution-And-Clock-Trust]]
- [[DALI-Telemetry-And-D4i]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Fail-Safe-Lighting-Rule]]

## Sources

- Research synthesis, 2026-09-09. PNNL-32949: https://www.pnnl.gov/main/publications/external/technical_reports/PNNL-32949.pdf · autonomous astro-calendar control as commercial norm: https://intelilight.eu/intelligent-street-lighting-control/ · CLO: https://www.ligman.com/clo-constant-lumen-output-and-night-dimming/ · DPWH spec summary (secondary): https://www.sresky.com/dpwh-road-lighting-specifications-why-are-high-lumen-output-and-typhoon-resistance-key-metrics-for-philippine-government-projects/

## History

- 2026-09-09: written with schedules in the slave gateways "and ideally in the driver devices".
- 2026-10-14: per-pole devices now hold a fallback schedule; group dimming moved to radio group commands with `execute_at`.

Back to [[Index]]
