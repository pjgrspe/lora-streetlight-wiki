# Street Lighting Control Logic

## Photocell vs astronomical clock

| | Photocell | Astronomical clock |
|---|---|---|
| Trigger | Measured ambient lux | Computed sunrise/sunset from lat/long + date |
| Failure mode | **Day-burner** (dirty/failed cell → on 24 h) or dark street | Clock drift, wrong coordinates, wrong timezone |
| Handles storms | **Yes** — under typhoon rain or ash fall, ambient light can legitimately drop to where lights *should* come on | **No** — stays off through a dark afternoon |
| Consistency | Poor — 128 cells switch at 128 slightly different times: ragged switch-on, staggered inrush | Excellent — all switch on one computed schedule |
| Energy predictability | Poor | Good |

**Recommendation: astronomical clock as primary, with a photocell (or a few reference light sensors) as an *override-on* input only — never override-off.** This is the standard architecture in commercial controllers. The computation belongs in the gateway, not the server — see [[Time-Distribution-And-Clock-Trust]] for the algorithm and Manila's figures.

## Dimming profiles

1. **Full-night constant** — baseline.
2. **Midnight / part-night dimming** — e.g. 100% until 00:00, 50% until 05:00, 100% to dawn.
3. **Multi-step adaptive** — 100/70/50/70% keyed to traffic bands. **Best expressed as DALI scenes** so one frame changes the whole street ([[DALI-Fundamentals]]).
4. **Part-night switch-off** — **do not do this on public roads** without an explicit safety and liability decision by the LGU.
5. **CLO (Constant Light Output)** — the driver holds lumens constant over life by starting at reduced current and ramping up as the LED depreciates. For L80@16 years that means starting **20% dimmed**, adding 20 points of power linearly over 16 years.

   ⚠ In a D4i system CLO can live in the driver *or* in the gateway (using Part 253 light-source on-time). **Pick one — running both double-compensates.**

## ⚠ Measured savings are well below the modelled figures

PNNL-32949 measured, against expectations:

| Strategy | Expected saving | **Measured** |
|---|---|---|
| Part-night dimming | 18% | **12%** |
| CLO L90@20yr | 5% | **≈0%** |
| CLO L80@16yr | 10% | **−2% (an increase in cost)** |

PNNL's recommendation: **calibrate the controller to each luminaire's measured dimming curve; do not assume linearity or standards compliance.**

> **Implication for the gateway: measure energy, don't model it.** The server should compare **commanded level → predicted power → measured power** and treat divergence as both a fault signal *and* a calibration input. See [[Energy-Metering-And-Telemetry-Contract]].

## Energy math for this system

128 luminaires × 100 W input, 12.0 h average night:

| Scenario | Nightly kWh | Annual kWh | @ ₱11/kWh |
|---|---|---|---|
| HPS baseline (250 W HPS ≈ 275 W input) | 422 | 154,200 | ₱1,696,000 |
| **LED 100%, all night** | 153.6 | 56,100 | ₱617,000 |
| LED + midnight dim | 115.2 | 42,100 | ₱463,000 |
| LED + 4-step adaptive (~35% reduction) | 99.8 | 36,400 | ₱401,000 |

LED retrofit alone is **~64% saving** versus HPS. Dimming adds a nominal 25% more — **but apply the PNNL de-rating and expect ~17% realised** if the drivers' dim response is not calibrated.

> **⚠ And the commercial precondition: if the connection is an unmetered "flat streetlight" service, dimming saves zero pesos.** See [[Philippines-Site-Context]] — this must be answered before dimming logic is worth building at all.

## The rule that shapes the whole design

**The nightly on/off must cost zero radio traffic in the normal case.** The schedule lives in the slave gateways (and ideally in the driver devices), so a congested, jammed or dead network cannot prevent the lights coming on. The radio is for supervision and override, not for the routine switch.

This is why [[Slave-Gateways-As-Intelligent-Sub-Controllers]] and [[Fail-Safe-Lighting-Rule]] are the two load-bearing decisions in the architecture.

## Legal floor on dimming depth

⚠ The **DOE Roadway Lighting Guidelines (2017)** and **DPWH DO No. 19** set illuminance in lux by road class (secondary sources give e.g. Class C feeder roads ≥10–15 lux with uniformity ≥0.3, and CCT 2,000–6,500 K with warm white required on arterials). **The primary DOE PDF was not retrievable** — the host failed DNS.

**Your MIN LEVEL and dimming profiles must not take illuminance below the mandated lux and uniformity for the road class. That is a legal constraint, not a preference.** Obtain the official documents before fixing dim floors. Tracked in [[Open-Questions]].

## Related

- [[Time-Distribution-And-Clock-Trust]]
- [[DALI-Telemetry-And-D4i]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. PNNL-32949: https://www.pnnl.gov/main/publications/external/technical_reports/PNNL-32949.pdf · autonomous astro-calendar control as commercial norm: https://intelilight.eu/intelligent-street-lighting-control/ · CLO: https://www.ligman.com/clo-constant-lumen-output-and-night-dimming/ · DPWH spec summary (secondary): https://www.sresky.com/dpwh-road-lighting-specifications-why-are-high-lumen-output-and-typhoon-resistance-key-metrics-for-philippine-government-projects/
