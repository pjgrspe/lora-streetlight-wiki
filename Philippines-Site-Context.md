# Philippines Site Context

Non-radio local realities that change the design. (Radio regulation is separate — see [[Philippines-Regulatory-Context]].)

## Grid

- **60 Hz, not 50.** This trips up designers importing European equipment — verify meter and driver frequency ratings.
- **230 V** nominal L-N per the Philippine Electrical Code; **400 V** L-L three-phase wye (400/230 V) for commercial/industrial.
- **Design for 230 V ±10% (207–253 V) as normal.** LV voltage excursion is significant, especially at feeder ends. Specify drivers for wide input (100–277 V or 120–277 V). **Log V per phase — it is a leading indicator of driver failures**, and it correlates directly with the Part 253 supply under/over-voltage counters.

> **⚠ Verify the actual service configuration on site.** Legacy Meralco three-phase secondary is frequently **230 V three-wire (delta / open-delta), not 400/230 V wye.** If the meter and the one-meter-per-phase topology assume a 4-wire wye with neutral but the real service is 3-wire delta, **your L-N voltage measurements and per-phase power calculations will be wrong.** This must be confirmed before the meter registers are interpreted. Tracked in [[Open-Questions]].

## The billing model decides whether the project's ROI exists

Meralco distinguishes two street-lighting services:

| Service | Fixtures owned by | Billing |
|---|---|---|
| **Flat streetlights** | Meralco | **Fixed assumed consumption per fixture type — i.e. unmetered** |
| **Metered streetlights** | Customer | **Actual consumption** |

> **This is the most commercially important finding in the research. On a flat/unmetered service, dimming and CLO produce zero billing savings** — the utility bills a fixed figure per fixture regardless of what the lights actually draw. The project must be on a **metered** service (customer-owned fixtures) for energy savings to convert to money.
>
> It also means the 3 phase meters are **the customer's own instrumentation, not revenue meters** — which relaxes the accuracy requirement to Class 1 and removes any need for utility-approved or sealed meters. See [[Energy-Metering-And-Telemetry-Contract]].

**Confirm which service class applies before sizing the savings case** — and therefore before [[Street-Lighting-Control-Logic]]'s dimming logic is worth building.

Tariff: Meralco ≈ **₱11–12/kWh**; provincial electric cooperatives ≈ **₱8–14/kWh**. Use the actual co-op tariff, not a national average.

## Regulatory (energy, not radio)

- **RA 11285 (Energy Efficiency and Conservation Act)**, approved 12 April 2019; IRR effective 21 December 2019 (DOE DC 2019-11-0014). DOE is lead agency, and **DPWH, coordinating with DOE, implements the Roadway Lighting Guidelines.**
- An **IAEECC resolution (May 2021)** directs all government agencies **including LGUs** to use energy-efficient LED lamps as a GEMP compliance requirement.
- **DOE Roadway Lighting Guidelines (2017)** and **DPWH DO No. 19** set lux, uniformity and CCT by road class — and therefore **legally bound how deep you may dim**. ⚠ The primary DOE PDF was not retrievable (host DNS failure); the figures circulating are from secondary sources. **Obtain the official documents before fixing dim floors.**

## Environmental realities

**Lightning.** Among the highest ground flash densities in Southeast Asia. A street-lighting run is a kilometres-long conductor with poles acting as strike attractors — **it is a lightning collector by construction.** Design per IEC 62305. Full mitigation detail in [[Enclosure-Power-And-Surge]]; the essentials are Class II SPDs at the cabinet, protection on the **DALI and RS485 pairs as well as mains**, a coax arrester at the antenna bulkhead, and single-point earthing with a measured **5 Ω target**.

**Typhoon.** Wind loading on luminaires and pole-top nodes, water ingress, extended outages. IP66 and IK08+ for outdoor enclosures; DPWH specs for government projects explicitly weight typhoon resistance.

**Corrosion.** Coastal salt spray → stainless fasteners, marine-grade coatings, isolated dissimilar metals.

**Heat and humidity.** 22–36 °C ambient with 70–90% RH year-round, and 60–80 °C inside a sealed pole-top or cabinet enclosure. **Expect real Part 253 thermal-derating events on west-facing poles in April–May.** Condensation, not water ingress, is the actual killer.

**Power reliability.** Frequent outages and deep sags. Three design consequences:

1. **All schedules and the astronomical clock must live in the gateway with a backed-up RTC** — not in the server, not in the cloud. See [[Time-Distribution-And-Clock-Trust]].
2. **DALI `POWER ON LEVEL` determines behaviour on restoration.** Leave it sane (100%) so a 2 a.m. mains restore lights the street immediately — see [[Fail-Safe-Lighting-Rule]].
3. **Expect the Part 253 driver start counter to climb fast. Don't misread that as a fault.**
4. UPS both the master gateway and the monitoring server.

## Operational note

Commissioning 128 DALI addresses is a night-time street exercise unless you pre-address on the bench. The **pre-addressing plus QR-scan-at-install** approach converts it to a daytime task — see [[Commissioning-And-Field-Swap]].

## Related

- [[Enclosure-Power-And-Surge]]
- [[Street-Lighting-Control-Logic]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Meralco flat vs metered streetlights: https://company.meralco.com.ph/faq-about/start-or-modify-service · RA 11285: https://elibrary.judiciary.gov.ph/thebookshelf/showdocs/2/90192 · IRR: https://www.pe2.org/sites/default/files/DOE-DC-2019-11-0014-RA11285-IRR-approved-20191122-OPT.pdf · secondary supply voltages: https://electricianphilippines.com/meralco-secondary-supply-voltages-230v-460v-400-230v/ · tariffs: https://www.voltflow.net/blog/electricity-cost-philippines-2026
