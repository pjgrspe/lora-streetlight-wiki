# DALI Bus Length Constraint

> **Likely resolved 2026-10-14.** The revised architecture points to **one LoRa LED driver/monitoring device per pole** (the "per-pole controller" branch of the fork below). Each DALI run then stays inside its own pole, and the 300 m limit stops binding. The radio carries the distance instead, with the costs described in [[Slave-Cell-Capacity-And-Scheduling]]. Pending client confirmation: [[Open-Questions]] Q3.

**The architecture as diagrammed — one DALI bus per section, serving 53/38/37 poles — is not physically buildable.** This is the most consequential downstream finding, because it determines what an "address" means in the gateway's data model.

## Correcting a common assumption

It is natural to assume 53 lights breaks DALI's 64-address limit. **It does not**: 53 ≤ 64, and 53 × 2 mA = 106 mA ≤ 250 mA. **Neither addressing nor bus current is the binding constraint anywhere in this system.**

## What actually binds: distance

DALI Alliance guidance is **300 m maximum between any two devices** on a bus, at 1.5 mm² and 250 mA. The derivation: ≤2 V drop over the pair at 250 mA → ≤~4 Ω per conductor; 1.5 mm² Cu at ~12 mΩ/m gives ~330 m.

**53 poles at typical 30–40 m roadway spacing is 1.6–2.1 km of run — 5x to 7x over the guidance.**

Note the rule is *between any two devices*, not total length, so a centre-fed bus with 150 m each way is legal — but that only buys ~300 m of street, i.e. **8–10 poles**. At very long lengths, cable capacitance becomes the limit through edge-rate distortion.

## Two further traps

**Controller channel derating.** Several commercial "broadcast master" gateways derate to **32 control gear per channel** because their integrated bus PSU is undersized. **Check the bus-PSU current rating of the actual controller, not the standard.**

**D4i integrated bus supplies collide.** If the drivers are D4i, each has an integrated bus PSU per Part 250 — some supply **up to 60 mA each**. So **more than four D4i drivers on one bus exceeds the 250 mA total, and you must disable the integrated supply on all but one.** Part 250 standardises the enable/disable for exactly this reason, but the default state is vendor-defined. **This is a classic field failure when D4i drivers are daisy-chained.**

## Options, ranked

| Option | How | Verdict |
|---|---|---|
| **1. Per-pole D4i controller (intra-luminaire DALI)** | Each luminaire has a Zhaga Book 18 or ANSI C136.41 socket; a radio node plugs in and talks DALI *inside* the luminaire. **The DALI bus is ~30 cm long; the radio does the distance.** | **Recommended.** Solves distance, gives per-light addressing, per-light energy and diagnostics, and real per-light failure identity. This is what the commercial industry does. Cost: 128 radios, not 3. |
| **2. Multiple DALI buses per section controller** | A multi-channel DALI controller (2/4/8 channels), each channel ≤300 m. 53 poles → ~6 channels or 6 controllers per section. | Viable if you must keep 3 radio nodes, but needs ~6 channels + 6 bus PSUs per section plus a DALI cable run down the whole street. **Cable cost usually kills it.** Address becomes a 3-tuple: (section, channel, short address). |
| **3. Broadcast-only** | One broadcast frame sets all gear. No commissioning, no addressing, no per-light feedback. | Only as a **degraded/fallback mode** — and it does not fix the distance problem. Keep it in the command set as an emergency "all to level X": it is the fastest, most robust command you have (one 15.83 ms frame, no reply). |
| **4. Groups** | 16 groups per bus, assigned by road segment or side of street. One frame commands many lights. | **Use this regardless of topology** — the bandwidth-efficient way to express "dim the north side to 60%", and with scenes you get 16 whole-street presets at one frame each. |

## What this means for the gateway — act on it now

Option 1 and option 2 imply *completely different* device counts on the radio network (128 nodes vs 6). **The gateway must not bake either in.** Make the address a hierarchical logical identifier independent of DALI:

```
site / section(1..3) / bus_or_node / dali_short_addr(0..63)  →  pole_id (GPS-tagged)
```

Support **four target types** in the command set — `broadcast`, `group(0..15)`, `short(0..63)`, `logical_pole_id` — and let the gateway resolve the last to whatever the physical topology turns out to be.

> **Do this and switching from option 2 to option 1 mid-project is a mapping-table change, not a protocol change.** Given how much of this project is still undecided, that is worth more than any other data-model decision.

⚠ **But note the tension:** option 1 puts 128 nodes on the radio, which is exactly the scenario [[LoRa-For-Control-Critique]] warns about and [[Media-Access-Control-Options]] shows needs ≥5-minute reporting under ALOHA. If option 1 is chosen, the MAC must be scheduled TDMA across multiple channels. **This is the biggest architectural fork in the project** — tracked in [[Open-Questions]].

## Related

- [[DALI-Fundamentals]]
- [[System-Architecture-As-Diagrammed]]
- [[Commissioning-And-Field-Swap]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. DALI Alliance cable-length tech note: https://www.dali-alliance.org/tech-notes/cable-length.html · D4i bus supply app note: https://www.inventronics-co.com/wp-content/uploads/2021/11/Application-Note-DALI-Bus-Supplies-and-Auxiliary-Supplies-Rev.-A.pdf · https://support.zencontrol.com/hc/en-us/articles/10310600966671-Maximum-Dali-Cable-Lengths
