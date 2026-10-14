# Enclosure, Power And Surge

A street-lighting cabinet is one of the worst surge and humidity environments in low-voltage distribution: long overhead feeders, tall metal poles acting as strike attractors, a mounted antenna, and a tropical day/night thermal cycle. In the Philippines this is not a footnote — **surge and condensation are the two most likely causes of a gateway dying in the field.**

## Mains AC-DC

- **Recommended: Mean Well `IRM-10-5`** — 5 V/2 A, **85–305 VAC input**, PCB-mount, encapsulated, no-load <0.1 W. The wide input matters where 230 V nominal sags and swells hard on a feeder shared with lighting contactors. DIN-rail alternative: `HDR-15-5`.
- Then a local 3.3 V regulator. At ~40 mA an LDO's 68 mW is irrelevant and it is **quieter for RF** than a buck.
- **Do not use HLK-PM01-class modules** for permanent outdoor mains installation — no surge rating, thin isolation margins, no relevant safety approvals. Fine on a bench, a liability in a roadside cabinet.
- ⚠ Mean Well does not publish IEC 61000-4-5 surge immunity in the IRM summary specs. **Assume it is inadequate alone** and provide external protection.

## Surge protection

**Mains side, staged per IEC 61643-11:**

1. **Class II SPD at the cabinet incomer** — L-N, L-PE, N-PE, `Up ≤ 1.5 kV`, `In` 5–20 kA. Use a **pluggable module with a status flag** so a technician can see a spent SPD.
2. Coordinated point-of-use on the AC-DC input: fuse → **thermally-protected MOV** (e.g. `TMOV14R275E`) L-N + a GDT N-PE. **The thermal protection matters:** a plain MOV degraded by repeated surges eventually conducts continuously and *catches fire*.
3. X2/Y2 filter caps and a common-mode choke after the fuse.

**Antenna/coax side, per IEC 61643-21:**

- Inline **coaxial GDT arrester at the cabinet bulkhead**. GDTs suit RF because of high surge current capability with very low capacitance and high off-state impedance.
- **Bond the arrester body to the cabinet ground bar with a short, thick strap — keep the ground lead under 0.5 m.** Long thin ground leads are inductive and defeat the arrester entirely.
- Bond a pole/mast to the *same* electrode as the cabinet. **Single-point ground is non-negotiable** — two separate earths with a coax between them is a lightning current path *through your PCB*.

**RS-485 and DALI:** both need surge protection, not just ESD diodes — see [[RS485-Hardware-Design]]. An unprotected long DALI bus is an antenna, which is another argument against inter-luminaire DALI here ([[DALI-Bus-Length-Constraint]]).

**Grounding:** single-point star at a cabinet ground bar. Target ≤10 Ω, and **5 Ω in high-lightning areas — which the Philippines is.** Measure and record it at commissioning.

## Thermal

Philippine ambient is 22–36 °C, but **solar gain in a closed roadside cabinet adds 20–35 °C**, and the cabinet often also holds contactors. **Design for 70–75 °C internal, 80 °C worst case.**

Therefore: **105 °C-grade MCU (suffix `7`)**, industrial-grade NOR and FRAM, X7R/X5R ceramics with DC-bias derating checked at temperature, and either 105 °C electrolytics rated ≥5000 h **or, better, polymer/ceramic only** on the low-voltage side. **Electrolytic caps are the number-one cause of "the gateway died after 18 months."**

Heat also degrades crystal accuracy ([[Time-Distribution-And-Clock-Trust]]) and LoRa sensitivity.

## Humidity is the real killer, not water ingress

Philippine RH is 70–90% year-round. A sealed cabinet with a day/night thermal cycle **condenses internally, every night, forever.** Do all of these:

- A **Gore-type pressure-equalising vent** rated to preserve IP66 — the single most effective measure.
- A drain path at the bottom; mount the PCB vertically.
- **Conformal coating** (acrylic or urethane), masking connectors and the antenna feed.
- Consider a small thermostatically-switched anti-condensation heater if the site has a history.

**IP65 minimum, IP66 preferred**, UV-stabilised polycarbonate/GRP or powder-coated metal. Note a metal enclosure **shields your radio** — an external antenna via a bulkhead connector is mandatory, which is also why the coax arrester belongs at the bulkhead.

**Corrosion:** coastal salt spray. Stainless or plated hardware, and avoid dissimilar-metal contact at the antenna mount — aluminium mast plus steel bolt gives galvanic corrosion *and* a rising RF ground impedance.

## Coax loss — usually larger than people assume

| Cable | ~loss @900 MHz | 3 m | 10 m |
|---|---|---|---|
| RG-174 | 1.0 dB/m | 3.0 dB | 10 dB |
| RG-58 | 0.6 dB/m | 1.8 dB | 6 dB |
| RG-316 / LMR-195 | 0.34 dB/m | 1.0 dB | 3.4 dB |
| **LMR-240** | 0.23 dB/m | 0.7 dB | 2.3 dB |
| **LMR-400** | 0.13 dB/m | 0.4 dB | 1.3 dB |

> On a link where the whole PA gives +22 dBm, **spending 3 dB on RG-58 throws away half your transmit power** — and the same 3 dB on receive is worse, because it degrades sensitivity directly.

Rules: ≤1 m runs, RG-316 is fine; **>2 m, LMR-240 minimum, LMR-400 for masts**. Every connector pair costs 0.1–0.3 dB — minimise adapters. **Use N-type, not SMA, outdoors**: SMA is not weatherproof and its small mating surface corrodes. **Weatherproof every outdoor connector** (self-amalgamating tape then vinyl) — water in coax raises loss catastrophically and gradually, presenting as "the link got worse over six months".

## EMC and compliance

**CISPR 15 / EN 55015** (emissions) and **EN 61547** (immunity) apply to lighting control gear, plus IEC 61000-4-2 (ESD), -4-4 (EFT), -4-5 (surge), -4-6 (conducted RF), and **-4-11 (dips/interruptions — worth testing, since lighting feeders dip)**. Radio type approval is NTC's, per [[Philippines-Regulatory-Context]].

**Layout:** radio RF section on a solid ground plane with a via fence; the RS-485 isolation barrier as a real physical gap (≥5 mm creepage, no traces or planes crossing under the isolator); the AC-DC primary in its own corner with a copper slot; RS-485 and mains routed away from the antenna feed; ferrite beads on the RS-485 pair at the connector.

## Related

- [[Link-Budget-And-Site-Survey]]
- [[Philippines-Site-Context]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. IRM-10-5: https://www.digikey.com/en/products/detail/mean-well-usa-inc/IRM-10-5/7704657 · IEC 61643-11: https://lsp.global/iec-61643-11-ac-surge-protector-standard-performance-safety/ · coax GDT arrestors: https://citel.us/en/gas-tube-surge-arrestors-for-coaxial-line-protection · IEC 62305 Metro Manila worked example: https://filipinoengineer.com/blog/2025/11/determination-and-calculation-of-lightning-protection-levels-lpl-based-on-iec-62305-with-application-to-an-industrial-building-in-metro-manila-philippines.html. ⚠ Coax loss figures are typical manufacturer values — confirm against the specific cable datasheet.
