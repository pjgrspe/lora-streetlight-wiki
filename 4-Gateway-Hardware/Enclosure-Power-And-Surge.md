# Enclosure, Power And Surge

How the gateway survives a Philippine roadside cabinet. **Surge and condensation are the two most likely causes of a gateway dying in the field.** The design responses are staged surge protection on every conductor that leaves the box, single-point earthing, a pressure-equalising vent, and no electrolytic capacitors where avoidable.

## Key points

- **Mains:** Mean Well `IRM-10-5` (85–305 VAC), then a quiet LDO to 3.3 V. No HLK-PM01-class modules outdoors.
- **Surge, staged:** Class II SPD at the incomer, a thermally-protected MOV at the supply, a coax GDT at the antenna bulkhead with a **ground strap under 0.5 m**, and protection on RS485 too.
- **Single-point earthing**, target **5 Ω** (high-lightning area). Measure and record it at commissioning.
- **Design for 70–75 °C internal (80 °C worst case).** Use 105 °C parts and polymer/ceramic capacitors.
- **Humidity:** a Gore-type vent, conformal coating, vertical PCB with a drain path. A sealed cabinet condenses every night.
- These items are now numbered requirements in spec 06 (HW-60…HW-94), each with a verification method.
- **Multiple radios mean multiple antennas and coax runs.** Each radio has its own N-type bulkhead, arrester and LMR-240+ run, with the antennas as far apart as practical (vertical stacking is best). ⚠ In-box isolation (~17–28 dB estimated) still isn't enough for simultaneous TX/RX, so measure it ([[Dual-Radio-Redundancy-And-Coexistence]]).

## Mains AC-DC

- **Recommended: Mean Well `IRM-10-5`**: 5 V/2 A, 85–305 VAC input, PCB-mount, encapsulated, no-load <0.1 W. The wide input matters on feeders shared with lighting contactors. DIN-rail alternative: `HDR-15-5`.
- Then a local 3.3 V regulator. An LDO is **quieter for RF** than a buck. ⚠ Size it for the radio count: an SX1276 draws ~10 mA in RX and far more transmitting at high power. A Slave has 4 radios, on **separately fused rails or LDOs per radio of a redundant pair**, so one failed radio can't drag down its backup.
- **Do not use HLK-PM01-class modules** for permanent outdoor mains: no surge rating, thin isolation, no relevant safety approvals.
- ⚠ Mean Well does not publish IEC 61000-4-5 surge immunity in the IRM summary specs. **Assume it is inadequate alone.**

## Surge protection

**Mains side, staged per IEC 61643-11:**

1. **Class II SPD at the cabinet incomer**: L-N, L-PE, N-PE, `Up ≤ 1.5 kV`, `In` 5–20 kA. Use a pluggable module with a status flag so a technician can see a spent SPD.
2. Point-of-use on the AC-DC input: fuse → **thermally-protected MOV** (e.g. `TMOV14R275E`) L-N + a GDT N-PE. A plain MOV degraded by repeated surges eventually conducts continuously and can catch fire.
3. X2/Y2 filter caps and a common-mode choke after the fuse.

**Antenna/coax side, per IEC 61643-21:**

- An inline **coaxial GDT arrester at the cabinet bulkhead, one per antenna**. GDTs suit RF: high surge capability, very low capacitance.
- **Bond each arrester to the cabinet ground bar with a short, thick strap, under 0.5 m.** A long thin lead is inductive and defeats the arrester.
- Bond the pole/mast to the *same* electrode. **Single-point ground is non-negotiable**: two earths with coax between them route lightning current through the PCB.

**RS-485:** surge protection, not just ESD diodes ([[RS485-Hardware-Design]]).

**Grounding:** single-point star at a cabinet ground bar. Target ≤10 Ω, and **5 Ω in high-lightning areas, which the Philippines is.**

## Thermal

Ambient is 22–36 °C, but **solar gain in a closed cabinet adds 20–35 °C**, often alongside contactors. **Design for 70–75 °C internal, 80 °C worst case.**

So: a **105 °C-grade MCU** on the production board, industrial-grade NOR/FRAM, X7R/X5R ceramics derated for DC bias at temperature, and **polymer/ceramic capacitors only** on the low-voltage side (or 105 °C, ≥5000 h electrolytics). **Electrolytics are the number-one cause of "the gateway died after 18 months."** Heat also degrades crystal accuracy ([[Time-Distribution-And-Clock-Trust]]) and receiver sensitivity.

## Humidity

Philippine RH is 70–90% year-round. A sealed cabinet on a day/night cycle **condenses internally every night.** Do all of these:

- A **Gore-type pressure-equalising vent** that preserves IP66: the single most effective measure.
- A drain path at the bottom; mount the PCB vertically.
- **Conformal coating** (acrylic or urethane), masking connectors and antenna feeds.
- A thermostatic anti-condensation heater if the site has a history.

**IP65 minimum, IP66 preferred**, UV-stabilised polycarbonate/GRP or powder-coated metal. A metal enclosure shields the radios, so **external antennas via bulkhead connectors are mandatory**. **Corrosion:** coastal salt spray. Use stainless/plated hardware and avoid dissimilar metals at antenna mounts.

## Coax and connectors

| Cable | ~loss @900 MHz | 3 m | 10 m |
|---|---|---|---|
| RG-174 | 1.0 dB/m | 3.0 dB | 10 dB |
| RG-58 | 0.6 dB/m | 1.8 dB | 6 dB |
| RG-316 / LMR-195 | 0.34 dB/m | 1.0 dB | 3.4 dB |
| **LMR-240** | 0.23 dB/m | 0.7 dB | 2.3 dB |
| **LMR-400** | 0.13 dB/m | 0.4 dB | 1.3 dB |

With EIRP capped at **16 dBm**, every dB of cable is a dB of link margin, and on receive it degrades sensitivity directly.

- Runs ≤1 m: RG-316 is fine. **>2 m: LMR-240 minimum; LMR-400 for masts.**
- Each connector pair costs 0.1–0.3 dB, so minimise adapters.
- **N-type outdoors, not SMA.** SMA is not weatherproof and corrodes.
- **Weatherproof every outdoor connector** (self-amalgamating tape, then vinyl). Water in coax raises loss gradually and presents as "the link got worse over six months."
- **Antenna gain counts toward EIRP.** Set PA output in firmware so PA + antenna gain − cable loss ≤ 16 dBm ([[Philippines-Regulatory-Context]]). The installer records each radio's antenna gain and cable loss; the firmware sets conducted power from them (spec 06 HW-84).
- Antennas of one box as far apart as practical, vertical stacking preferred, ≥1 m target. The final spacing comes from the in-box isolation test in spec 06 §11.

## EMC and compliance

**CISPR 15 / EN 55015** (emissions) and **EN 61547** (immunity) apply to lighting control gear, plus IEC 61000-4-2 (ESD), -4-4 (EFT), -4-5 (surge), -4-6 (conducted RF) and **-4-11 (dips; lighting feeders dip)**. Radio type approval is NTC's ([[Philippines-Regulatory-Context]]).

**Layout:** RF section on a solid ground plane with a via fence; the RS-485 isolation barrier as a real gap (≥5 mm creepage, nothing crossing under the isolator); the AC-DC primary in its own corner with a copper slot; RS-485 and mains routed away from antenna feeds; ferrite beads on the RS-485 pair at the connector.

## Related

- [[RS485-Hardware-Design]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Link-Budget-And-Site-Survey]]
- [[Philippines-Site-Context]]

## Sources

- `Specs/specs/06-gateway-hardware-requirements.md` §7–§11 (v0.1, 2026-10-21).
- Research synthesis, 2026-09-09. IRM-10-5: https://www.digikey.com/en/products/detail/mean-well-usa-inc/IRM-10-5/7704657 · IEC 61643-11: https://lsp.global/iec-61643-11-ac-surge-protector-standard-performance-safety/ · coax GDT arrestors: https://citel.us/en/gas-tube-surge-arrestors-for-coaxial-line-protection · IEC 62305 Metro Manila worked example: https://filipinoengineer.com/blog/2025/11/determination-and-calculation-of-lightning-protection-levels-lpl-based-on-iec-62305-with-application-to-an-industrial-building-in-metro-manila-philippines.html. ⚠ Coax loss figures are typical manufacturer values; confirm against the specific cable datasheet.

## History

- 2026-09-09: Written for a single-radio gateway.
- 2026-10-14: Updated for multiple radios per box (one antenna/coax/arrester each, per-radio rails) and the decided 16 dBm EIRP cap. The gateway hardware itself is built by the separate hardware developer.
- 2026-10-21: aligned with spec pack v0.1 (requirements numbered in 06, EIRP recording rule, antenna spacing set by the isolation test).

Back to [[Index]]
