# Philippines Site Context

The non-radio local realities that change the design: the grid, the billing model, energy regulation and the environment. (Radio rules are in [[Philippines-Regulatory-Context]].)

## Key points

- **60 Hz, 230 V L-N**, and design for **±10% as normal**. Log voltage per phase — it predicts driver failures.
- ⚠ **Confirm wye vs delta on site.** Legacy Meralco three-phase secondary is often 230 V three-wire delta; per-phase power maths assumes wye.
- **The savings case only exists on a metered streetlight service.** On a flat service, dimming saves nothing.
- **Lightning and condensation** are the two most likely causes of field failures.
- Frequent outages mean **schedules and the astronomical clock live in the gateways and devices**, never in the server or cloud.

## Grid

- **60 Hz, not 50.** Verify meter and driver frequency ratings, especially on European equipment.
- **230 V** nominal L-N (Philippine Electrical Code); **400/230 V** three-phase wye for commercial/industrial.
- **Design for 207–253 V as normal.** LV excursion is significant at feeder ends. Specify drivers for wide input (100–277 V or 120–277 V). **Log V per phase** — it is a leading indicator of driver failures and correlates with the DALI Part 253 under/over-voltage counters.
- ⚠ **Service configuration:** if the meter and per-phase topology assume 4-wire wye but the real service is 3-wire delta / open-delta, **L-N voltages and per-phase power will be wrong**. Confirm before interpreting meter registers ([[Open-Questions]] Q9).

## The billing model decides whether the ROI exists

| Meralco service | Fixtures owned by | Billing |
|---|---|---|
| **Flat streetlights** | Meralco | **Fixed assumed consumption per fixture — unmetered** |
| **Metered streetlights** | Customer | **Actual consumption** |

**On a flat/unmetered service, dimming and CLO produce zero billing savings.** The project must be on a metered, customer-owned service for energy savings to become money ([[Open-Questions]] Q4). Confirm this before sizing the savings case or investing in the dimming logic in [[Street-Lighting-Control-Logic]].

It also means the section energy meters are **the customer's own instrumentation, not revenue meters** — Class 1 accuracy is enough, and no utility-approved or sealed meters are needed ([[Energy-Metering-And-Telemetry-Contract]]).

Tariff: Meralco ≈ **₱11–12/kWh**; provincial co-ops ≈ **₱8–14/kWh**. Use the actual co-op tariff.

## Energy regulation (not radio)

- **RA 11285 (Energy Efficiency and Conservation Act)**, approved 12 April 2019; IRR effective 21 December 2019 (DOE DC 2019-11-0014). DOE leads; **DPWH, with DOE, implements the Roadway Lighting Guidelines.**
- An **IAEECC resolution (May 2021)** directs all government agencies, **including LGUs**, to use energy-efficient LED lamps.
- **DOE Roadway Lighting Guidelines (2017)** and **DPWH DO No. 19** set lux, uniformity and CCT by road class — and so **legally bound how deep you may dim**. ⚠ The primary DOE PDF was not retrievable; circulating figures are secondary. Get the official documents before fixing dim floors ([[Open-Questions]] Q12).

## Environment

| Hazard | What it means for the design |
|---|---|
| **Lightning** | Among the highest ground flash densities in SE Asia; a street-lighting run is a lightning collector by construction. Design per IEC 62305: Class II SPDs at the cabinet, protection on **RS485 and DALI pairs as well as mains**, a coax arrester at the antenna bulkhead, single-point earthing at a measured **5 Ω**. See [[Enclosure-Power-And-Surge]] |
| **Typhoon** | Wind load on pole-top hardware, water ingress, long outages. IP66 and IK08+ outdoors |
| **Corrosion** | Coastal salt spray → stainless fasteners, marine coatings, isolated dissimilar metals |
| **Heat and humidity** | 22–36 °C ambient, 70–90% RH year-round, 60–80 °C inside sealed enclosures. Expect thermal-derating events on west-facing poles in April–May. **Condensation, not water ingress, is the real killer** |

## Power reliability

Frequent outages and deep sags. Consequences:

1. **Schedules and the astronomical clock live in the gateways and devices with a backed-up RTC** — not in the server. See [[Time-Distribution-And-Clock-Trust]].
2. **DALI `POWER ON LEVEL` = 100%**, so a 2 a.m. mains restore lights the street immediately ([[Fail-Safe-Lighting-Rule]]).
3. Expect the driver start counters to climb fast — not a fault.
4. Put the master gateway and the monitoring server on a UPS.

## Commissioning note

With one device per pole, commissioning is mainly **device-to-pole mapping**. Pre-configure on the bench and **QR-scan at install** to turn a night-time street exercise into a daytime task — see [[Commissioning-And-Field-Swap]].

## Related

- [[Philippines-Regulatory-Context]]
- [[Enclosure-Power-And-Surge]]
- [[Street-Lighting-Control-Logic]]
- [[Energy-Metering-And-Telemetry-Contract]]

## Sources

- Research synthesis, 2026-09-09. Meralco flat vs metered streetlights: https://company.meralco.com.ph/faq-about/start-or-modify-service · RA 11285: https://elibrary.judiciary.gov.ph/thebookshelf/showdocs/2/90192 · IRR: https://www.pe2.org/sites/default/files/DOE-DC-2019-11-0014-RA11285-IRR-approved-20191122-OPT.pdf · secondary supply voltages: https://electricianphilippines.com/meralco-secondary-supply-voltages-230v-460v-400-230v/ · tariffs: https://www.voltflow.net/blog/electricity-cost-philippines-2026

## History

- 2026-09-09 — written.
- 2026-10-14 — commissioning note updated for one device per pole; meters described per section rather than per phase.

Back to [[Index]]
