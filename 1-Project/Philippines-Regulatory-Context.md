# Philippines Regulatory Context (Radio)

The radio rules the system is designed to, and what is still unconfirmed. The project has **decided to design to the Philippine 915–918 MHz envelope**; the open risk is whether NTC requires a licence and what its exact limits are.

## Key points

- **Band: 915–918 MHz** (NTC allocation; LoRa Alliance AS923-3 for the Philippines). Not 868 MHz, not US 902–928 MHz, not 433 MHz.
- **Working profile:** 125 kHz channels on a 200 kHz raster (~15 channels), **≤16 dBm EIRP**, **≤400 ms per transmission** (SF7–SF10 only), **≤1% duty cycle accounted per box**.
- Band, channel plan and TX power are **configuration parameters** with a compile-time regional ceiling — a different NTC answer is a config change, not a rewrite.
- ⚠ **Not settled:** whether the band is licence-exempt, and NTC's confirmed EIRP/duty/dwell figures. **The letter to NTC is still required.**
- Type approval is mandatory and per RF module.

## The decided radio profile

| Parameter | Value |
|---|---|
| Band | 915–918 MHz (3 MHz) |
| Channel width / raster | 125 kHz on 200 kHz spacing, ~15 channels |
| Channel use | 2 tier-1 channels at opposite band edges + 2 per section ([[Dual-Radio-Redundancy-And-Coexistence]]) |
| Max EIRP | 16 dBm (antenna gain counts) |
| Max single transmission | 400 ms → SF7–SF10; SF11/SF12 unusable ([[LoRa-Airtime-Budget]]) |
| Duty cycle | ≤1%, ledger kept per box (multi-radio treatment unconfirmed — [[Open-Questions]] Q21) |

Because the channel plan grows by two per section, the section count ([[Open-Questions]] Q27) and the usable channel count are linked: roughly 6 sections fit before channels must be reused across distance.

## What is verified

**NTC National Radio Frequency Allocation Table (NRFAT-2016 Rev. 2)**, 890–942 MHz national column:

| Band | Philippine national use | Circular cited |
|---|---|---|
| 890–915 MHz | Cellular Mobile Telephone System | — |
| **915–916 MHz** | Point-to-Point Radio System | **MC 03-08-2013**, "Wireless Data Networks and Devices", 915–918 MHz |
| **916–918 MHz** | Point-to-Multipoint Radio System | same |
| 918–920 MHz | RFID, 25 kHz spacing | MC 03-08-2006 |
| 920–925 MHz | Broadcast Studio-Transmitter Link | — |
| 925–935 MHz | Cellular | — |

**LoRa Alliance RP002-1.0.3 Table 1** lists `Philippines (PH) | 915–918 MHz | AS923-3`. AS923-3 has a −6.60 MHz offset, so its default channels are **916.6 and 916.8 MHz**, 125 kHz, DR0–DR5, duty cycle <1%, default max EIRP **16 dBm**.

US-style 902–928 MHz is not available. 433 MHz exists under MC 03-05-2007 but at only **10 mW e.r.p.** (~10 dB handicap) and is shared with amateur radiolocation and ISM heating.

## What could not be verified ⚠

- **The full text of NTC MC 03-08-2013 was not retrievable**, so the real EIRP, duty cycle, dwell time, occupied bandwidth and — critically — **licence-exempt vs licensed status** are unknown.
- The NRFAT wording "Point-to-Point / Point-to-Multipoint **Radio System**" often implies a **licensed** service with frequency assignment in NTC usage. That would be a project-level problem.
- **868 MHz is contradictory in NTC's own documents** (MC 03-05-2007 lists 863–870 MHz SRD sub-bands; the NRFAT shows 866–869 MHz as trunked radio and 869–890 MHz as cellular). Do not design for it.
- The newest SRD amendment, **MC 004-06-2026**, covers only 2.4 GHz and above (movement-detection devices). No sub-GHz content.

## Duty cycle vs dwell time

| Regime | Rule | Consequence |
|---|---|---|
| **Duty cycle** (EU/ETSI style) | Time-averaged transmit fraction per hour, e.g. 1% | A 185 ms SF9 frame allows one transmission every 18.5 s per box |
| **Dwell time** (US/AU/JP style) | Hard cap on a *single* transmission — 400 ms in AS923 | **Removes SF11/SF12 entirely** |

The design assumes both apply. AS923 mandates listen-before-talk only for Japan; RP002 defers per-transmission rules to national regulation.

## The EIRP trap

Antenna gain counts. A +20 dBm SX1276 PA into a 5 dBi antenna is ~+25 dBm EIRP — **9 dB over a 16 dBm limit**. PA output is set in firmware so the link budget closes at the *legal* power, not the silicon's capability.

## Actions

1. **Write to NTC** — Equipment Standards Division, `equipment.standard@ntc.gov.ph` — asking in writing for: (a) the text of MC 03-08-2013; (b) whether 915–918 MHz LPWAN use is licence-exempt or needs a station licence / frequency assignment; (c) max EIRP, occupied bandwidth and duty-cycle/dwell rules; (d) whether AS923-3 is accepted as a compliance profile; (e) whether a multi-radio gateway has one duty-cycle budget or one per radio.
2. **Type approval** is mandatory, per RF module. NTC accepts host approval based on the specific integrated RF module; EU standards accepted (FCC sometimes); foreign entities need a local representative; ~4–8 weeks. ⚠ Check whether the chosen SX1276 module's certification is accepted ([[Open-Questions]] Q10).
3. Keep band, channel plan and TX power in configuration with a compile-time ceiling.

## Related

- [[LoRa-Airtime-Budget]]
- [[Link-Budget-And-Site-Survey]]
- [[Radio-Hardware-Selection]]
- [[Open-Questions]]

## Sources

- Research synthesis, 2026-09-09. NRFAT-2016 Rev. 2: https://region7.ntc.gov.ph/wp-content/uploads/2024/01/NRFAT-2016-Rev-2.pdf · RP002-1.0.3 Table 1 p.16 · NTC circulars index: https://ntc.gov.ph/memorandum-circulars/ · Type approval: https://www.csagroup.org/global-certification-regulatory-update/philippines-ntc-type-approval-acceptance-of-equipment-based-on-specific-radio-module/

## History

- 2026-09-09 — research; interim envelope 916.5–917.0 MHz core, ≤16 dBm EIRP, ≤1% duty, ≤400 ms.
- 2026-10-14 — decided: design to the full 915–918 MHz band with the AS923-3-style profile above; duty ledger per box; NTC letter still required for licensing and confirmed limits.

Back to [[Index]]
