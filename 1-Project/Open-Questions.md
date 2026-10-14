# Open Questions

Everything still unresolved, grouped by how much damage a wrong answer would do. The top group could each force a redesign and should be chased before hardware or firmware is frozen.

## Key points

- **Regulation is the biggest risk.** The radio profile is decided (915–918 MHz), but licensing status and NTC's confirmed limits are not — the NTC letter is still needed (Q1, Q21).
- **The commercial case depends on a metered electricity service** (Q4), and **the real link budget is only knowable from a site survey** (Q5).
- Section count (Q27) and exact device count are still open; the design is built for N sections and up to 60 devices per Slave.
- Q2, Q3, Q6, Q22 and Q25 were resolved on 2026-10-14 — see the end of the note.

Q numbers are stable; don't renumber.

## Open — could force a redesign

### Q1 — Philippine radio regulation: licensing and confirmed limits

**Decided:** design to 915–918 MHz, 125 kHz channels, ≤16 dBm EIRP, ≤400 ms per transmission, ≤1% duty cycle per box ([[Philippines-Regulatory-Context]]).

**Still unknown:** NTC MC 03-08-2013's text was not retrievable, so the confirmed EIRP/duty/dwell limits — and **whether the band is licence-exempt at all** — are unknown. The NRFAT labels it "Point-to-Point / Point-to-Multipoint **Radio System**", which in NTC usage often implies a *licensed* service. If so, that is a project-level problem.

**Resolve by:** writing to NTC Equipment Standards Division (`equipment.standard@ntc.gov.ph`) for the circular text, licensing status, EIRP/duty/dwell limits, and whether AS923-3 is accepted as a compliance profile. Add Q21's questions to the same letter.

### Q4 — Flat (unmetered) or metered electricity service?

On a Meralco/co-op **flat streetlight** service the utility bills a fixed figure per fixture, so **dimming saves zero pesos**. The energy-savings case — and whether the dimming logic in [[Street-Lighting-Control-Logic]] is worth building — depends on a metered service. → [[Philippines-Site-Context]]

### Q5 — What is the real link budget on site?

Everything assumes SF9 works. Model error alone is ±14 dB, comparable to the whole fade margin, and under a 400 ms dwell limit SF11/SF12 aren't available as a fallback. **Only a physical survey with the real radios at the real heights answers this**, ideally before the firmware architecture is frozen. → [[Link-Budget-And-Site-Survey]]

### Q21 — One duty-cycle budget per box or per radio? Are enough channels allowed?

A slave has 4 radios, and the channel plan needs 2 tier-1 channels + 2 per section — which grows with the section count. Ask NTC whether a multi-radio box gets one 1% budget or one per radio, and confirm the full 915–918 MHz band (~15 channels) is usable. **Meanwhile the ledger is per box** (conservative). → [[Dual-Radio-Redundancy-And-Coexistence]]

## Open — needed soon

### Q7 — Which existing DALI drivers, and what do they support?

The project uses existing off-the-shelf DALI drivers, so this is a verification question — answer before the register map is frozen:

- Make and model(s), and whether the population is homogeneous (mixed physical minima cause uneven dimming at the bottom of the range).
- Any D4i-certified? Decides whether Parts 251/252/253 (per-light energy, diagnostics) exist.
- Part 250 bus PSU behaviour and Part 150 24 V aux — decides whether the per-pole device needs its own supply.

**Design stance:** Parts 251/252/253 are optional enrichment; nothing in control, fault detection or fail-safe may depend on them, and capabilities are probed at commissioning. → [[DALI-Telemetry-And-D4i]], [[Energy-Metering-And-Telemetry-Contract]]

### Q8 — Can the LED driver device hold an autonomous fallback schedule?

Fail-safe layer 3 depends on it. Without it, a dead slave falls straight through to the DALI System Failure Level — uncontrolled 100% all night. Needs an RTC, lat/long and a schedule table in the device developer's firmware. → [[Fail-Safe-Lighting-Rule]]

### Q9 — Is the 3-phase service wye or delta?

Legacy Meralco secondary is often **230 V three-wire delta**, not 400/230 V wye. If the meters assume wye with neutral and the service is delta, L-N voltages and per-phase power are wrong. Confirm on site. → [[Philippines-Site-Context]]

### Q10 — Will NTC accept the radio module's existing certification?

Type approval is per RF module. Whether the chosen SX1276 module (e.g. RFM95W-915) carries certification NTC accepts decides whether local testing is needed. ⚠ Not yet checked for the SX1276 modules. **Worth months of schedule.** → [[Radio-Hardware-Selection]]

### Q12 — DOE / DPWH mandated lux and uniformity by road class

The primary DOE 2017 guidelines PDF was not retrievable; circulating figures are secondary. **This legally bounds how deep you may dim.** → [[Street-Lighting-Control-Logic]]

### Q13 — DALI-2 frame settling timings

Sources conflict (forward→backward 2.92–9.17 ms vs 5.5–10.5 ms; forward→forward ≥9.17 ms vs >13.5 ms). **Interim: wait the union** — receive window 2.9–11 ms, forward spacing ≥13.5 ms. → [[DALI-Fundamentals]]

### Q23 — SX1276 is "not recommended for new designs"

Semtech's own pages carry that status. Not end-of-life, but a supply risk across over a hundred radios. Recommend a buffer buy and keep the radio abstraction so SX1262 stays a port, not a rewrite. Make sure the client knows. → [[Radio-Hardware-Selection]]

### Q24 — Measured antenna isolation inside the real enclosure

The in-box coexistence rules assume ~20–30 dB isolation, far short of what simultaneous TX/RX needs. Measure on the real enclosure during bench radio work. → [[Dual-Radio-Redundancy-And-Coexistence]]

### Q27 — How many sections / Slave Gateways?

The diagram shows 3; the project owner says it may be 4 or more. Design for N: one slave firmware, a tier-1 slot schedule sized from the configured slave count, and a channel plan of 2 + 2 per section (depends on Q1/Q21). Per-section ceiling stays at 60 devices.

## Open — can wait

### Q15 — Outage buffer sizing

How long an outage must the gateways survive, and at what resolution? "7 days of full telemetry" needs external NOR. **Likely answer: downsample** — 15-minute summaries plus full resolution for the last hour. → [[Persistence-And-Storage]]

### Q16 — Physical distances between master, slaves and poles

Not stated anywhere; feeds directly into Q5 and the survey plan.

### Q17 — Is VBAT usable on the chosen board?

On Nucleo-64 boards VBAT is tied to 3V3 by default (solder bridge, UM1724 ⚠ verify). Some modules tie VBAT to VDD internally, forcing an external RTC. → [[Time-Distribution-And-Clock-Trust]]

### Q18 — Scale-out intentions

If the pilot grows to many more sections, spectrum (≈15 channels in 3 MHz) and the master's duty cycle become binding. Worth knowing now so the address space and register-map strides don't need renumbering. Related to Q27.

### Q19 — Security posture expectations

Is a security review in this asset's future? Retrofitting a MIC and frame counters after deployment will be skipped, so they are in the wire format from the start even if key management stays crude at pilot. → [[Custom-Protocol-Security]]

### Q26 — Real devices for bench integration

How many real meters and LED devices will the device developer supply, and when? The Statement of Work in `Output/` assumes at least 1 meter and ~6 LED devices by integration, with the rest of a 60-device section simulated.

## Resolved

| Q | Date | Outcome |
|---|---|---|
| Q2 | 2026-10-14 | Devices are in-house (device developer, STM32F103 + SX1276); the protocol is ours, delivered as a linkable device-role library |
| Q3 | 2026-10-14 | One LoRa LED driver & monitoring device per pole — the per-pole branch of the DALI topology fork |
| Q6 | 2026-10-14 | pgorospe writes the monitoring server, speaking Modbus ([[Local-Monitoring-Server-Design]]). Client PC/OS still to confirm |
| Q11 | 2026-10-14 | No longer relevant — STM32WLE5 dropped in favour of STM32 + external SX1276 |
| Q14 | 2026-10-14 | Duty-cycle ledger is kept per box (conservative); NTC confirmation tracked under Q21 |
| Q20 | 2026-10-14 | No longer relevant — SX1276 uses fixed-algorithm CAD, so the SX1262 CAD parameters don't apply |
| Q22 | 2026-10-14 | A separate gateway hardware developer, following client STM32 dev boards + SX1276 modules and pgorospe's hardware requirements spec |
| Q25 | 2026-10-14 | Each device has one radio |

## Things worth verifying against primary documents

- AN5554 FUOTA flash partitioning
- Modbus-over-serial spec: the 150 Ω termination, 450–650 Ω polarisation range, and 1000 m-at-9600 figures
- DALI-2 `System Failure Level` / `Power-On Level` exact semantics and the ~550 ms interruption threshold (secondary sources only)
- SX1276 blocking figures (secondary citation of the datasheet)
- Coax loss figures per metre (typical manufacturer values)
- Mean Well IRM series IEC 61000-4-5 surge immunity

## Related

- [[Development-Roadmap-And-Gates]] — the specification gate is where most of these should be closed
- [[Philippines-Regulatory-Context]]
- [[System-Architecture-As-Diagrammed]]

## History

- 2026-09-09 — log started with Q1–Q20.
- 2026-10-14 — Q2, Q3, Q6, Q22, Q25 resolved by the revised architecture; Q11, Q14, Q20 closed as superseded; Q21–Q27 added; radio profile decided under Q1.

Back to [[Index]]
