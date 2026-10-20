# DALI Bus Length Constraint

Why the system uses **one LoRa LED driver device per pole** instead of one DALI bus per section. A section-long DALI bus is not physically buildable, so each pole gets its own short DALI link and the radio covers the distance.

## Key points

- **The binding limit is distance, not addresses or current.** DALI guidance is 300 m between any two devices; a 53-pole section is 1.6–2.1 km.
- **Resolved toward per-pole:** each pole has its own LoRa LED driver & monitoring device driving a ~30 cm in-pole DALI link.
- Cost of the choice: **dozens of radio nodes per section** (up to 60), which forces scheduled TDMA on the radio. See [[Slave-Cell-Capacity-And-Scheduling]].
- Benefit: real **per-light addressing, fault identity and telemetry**.
- The hierarchical logical ID still applies, so the topology is a mapping-table detail, not a protocol detail.

## The arithmetic

**53 lights do not break DALI's limits:** 53 ≤ 64 addresses, and 53 × 2 mA = 106 mA ≤ 250 mA.

**Distance does.** DALI Alliance guidance: **300 m maximum between any two devices** at 1.5 mm² and 250 mA. Derivation: ≤2 V drop at 250 mA → ≤~4 Ω per conductor; 1.5 mm² Cu at ~12 mΩ/m → ~330 m.

**53 poles at 30–40 m spacing = 1.6–2.1 km: 5–7× over.** The rule is *between any two devices*, so a centre-fed bus buys only ~300 m of street (8–10 poles). At long lengths cable capacitance also distorts edges.

The 53/38/37 per-section counts are the diagrammed example; the number of sections is not fixed.

## The options that were weighed

| Option | How | Outcome |
|---|---|---|
| **1. Per-pole controller (in-luminaire/in-pole DALI)** | A radio device per pole talks DALI over ~30 cm; the radio does the distance | **Chosen (2026-10-14).** Per-light addressing, energy, diagnostics and failure identity. Commercial norm |
| 2. Multi-channel DALI controllers | ~6 channels of ≤300 m each per 53-pole section, plus a DALI cable down the whole street | Rejected: cable and bus-PSU cost, and still needs a DALI run along the road |
| 3. Broadcast-only | One frame sets all gear, no feedback | Doesn't fix distance. The "all to level X" idea survives as a radio-level group command |
| 4. Groups | 16 groups per bus | Now done in the radio protocol (group commands with `execute_at`) |

## Traps that still apply inside the pole

- **Bus power.** Plain DALI-2/DALI-1 drivers need a separate DALI bus supply; the per-pole device must provide it (device developer). ⚠ Confirm in the device design.
- **D4i integrated supplies collide.** D4i drivers each have a Part 250 bus PSU (some up to 60 mA). More than four on one bus exceeds 250 mA; disable all but one. Mostly irrelevant with one driver per pole, but matters if a pole carries several luminaires.
- **Controller derating.** Some commercial masters derate to 32 gear per channel because of small integrated PSUs. Check the actual hardware rating.

## Addressing in the data model

Use a hierarchical logical ID, independent of DALI:

```
site / section / device (pole) / dali_short_addr  →  pole_id (GPS-tagged)
```

Command targets are the 12-bit radio address of a device, a whole section, the whole site, or one of 16 groups (`0x8000 | g`); the server maps pole IDs to device addresses (spec 01 §8, spec 00 §3). Inside the pole the device addresses its driver by broadcast or a fixed short address. See [[Device-Identity-Namespaces]].

## Related

- [[DALI-Fundamentals]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[System-Architecture-As-Diagrammed]]
- [[Commissioning-And-Field-Swap]]

## Sources

- `Specs/specs/00` §3, `Specs/specs/01` (command targets), v0.1.
- Research synthesis, 2026-09-09. DALI Alliance cable-length tech note: https://www.dali-alliance.org/tech-notes/cable-length.html · D4i bus supply app note: https://www.inventronics-co.com/wp-content/uploads/2021/11/Application-Note-DALI-Bus-Supplies-and-Auxiliary-Supplies-Rev.-A.pdf · https://support.zencontrol.com/hc/en-us/articles/10310600966671-Maximum-Dali-Cable-Lengths

## History

- 2026-09-09: identified that one DALI bus per section cannot be built; per-pole vs multi-channel left as the project's biggest open fork.
- 2026-10-14: resolved per-pole, from the client's revised architecture (one device per pole confirmed by the project owner).
- 2026-10-21: aligned with spec pack v0.1 (command target types).

Back to [[Index]]
