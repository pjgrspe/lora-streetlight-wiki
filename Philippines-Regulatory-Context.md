# Philippines Regulatory Context (Radio)

**This is the highest-uncertainty, highest-consequence open item in the project.** It affects the antenna, the RF matching network, the PHY parameters, and the entire airtime budget. Resolve it before any hardware commit.

## What is verified

**NTC National Radio Frequency Allocation Table (NRFAT-2016 Rev. 2)**, 890–942 MHz Philippine national column:

| Band | Philippine national use | Circular cited |
|---|---|---|
| 890–915 MHz | Cellular Mobile Telephone System | — |
| **915–916 MHz** | Point-to-Point Radio System | **MC 03-08-2013**, "Wireless Data Networks and Devices", 915–918 MHz |
| **916–918 MHz** | Point-to-Multipoint Radio System | same |
| 918–920 MHz | RFID, 25 kHz spacing | MC 03-08-2006 |
| 920–925 MHz | Broadcast Studio-Transmitter Link | — |
| 925–935 MHz | Cellular | — |

**LoRa Alliance RP002-1.0.3 Table 1** lists `Philippines (PH) | 915–918 MHz | AS923-3`. AS923-3 carries a −6.60 MHz offset, so its two default channels are **916.6 and 916.8 MHz**, 125 kHz, DR0–DR5, duty cycle <1%, default max EIRP **16 dBm**.

**Conclusion on band:** the usable slice is **915–918 MHz — only 3 MHz**. US-style 902–928 MHz operation is not available. 868 MHz should not be designed for (see below). 433 MHz is available under MC 03-05-2007 but at only **10 mW e.r.p.**, a ~10 dB handicap, and is shared with amateur radiolocation and ISM heating.

## What could NOT be verified

- **The full text of NTC MC 03-08-2013 was not retrievable.** So the actual Philippine limits on max EIRP, duty cycle, dwell time, occupied bandwidth, and — critically — **whether operation is licence-exempt or requires a radio station licence** are all unknown.
- The NRFAT labels these bands "Point-to-Point / Point-to-Multipoint **Radio System**". In NTC usage that phrasing often implies a **licensed** service with frequency assignment, not a licence-exempt SRD band. **That would be a project-level problem, not a tuning detail.**
- **868 MHz is internally contradictory in NTC's own documents**: MC 03-05-2007 lists 863–870 MHz SRD sub-bands, while the NRFAT shows 866–869 MHz as Trunked Radio and 869–890 MHz as cellular. **Do not design for 868 MHz in the Philippines.**
- The newest SRD amendment, **MC 004-06-2026**, covers only 2.4 GHz and above and is scoped to movement-detection devices. **No sub-GHz content — it does not help.**

## Duty cycle vs dwell time — two different regimes

The distinction matters enormously:

| Regime | Rule | Consequence |
|---|---|---|
| **Duty cycle** (EU/ETSI style) | Time-averaged transmit fraction per hour per sub-band, e.g. 1% | A 185 ms SF9 frame permits one transmission every 18.5 s per node |
| **Dwell time** (US/AU/JP style) | Hard cap on a *single* transmission — 400 ms in AS923, with ~20 s recommended between uplinks | **Kills SF11/SF12 entirely** — see [[LoRa-Airtime-Budget]] |

AS923 mandates listen-before-talk only for Japan. RP002 explicitly defers per-transmission parameters to national rules.

## Actions before hardware freeze

1. **Write to NTC** — Equipment Standards Division, `equipment.standard@ntc.gov.ph` — asking in writing for: (a) the full text of MC 03-08-2013; (b) whether 915–918 MHz LPWAN operation is licence-exempt or needs a station licence/frequency assignment; (c) max EIRP, occupied bandwidth, and duty-cycle/dwell rules; (d) whether AS923-3 is accepted as a compliance profile.
2. **Assume the conservative envelope meanwhile**: 916.5–917.0 MHz core, 125 kHz, **≤16 dBm EIRP**, **≤1% duty cycle per node**, **≤400 ms per transmission**.
3. **Type approval is mandatory** in the Philippines, per RF module. NTC accepts host approval based on the specific integrated RF module, EU standards accepted (FCC sometimes), foreign entities need a local representative, ~4–8 weeks. **Using a pre-certified module is worth real money and schedule here** — see [[Radio-Hardware-Selection]].
4. **Make the band a compile-time/NVM parameter** so firmware can move the whole channel plan without a hardware respin. Make TX power a config register with a compile-time regional ceiling.

## The EIRP trap

If the limit is 25 mW e.r.p. (+14 dBm), then a +22 dBm PA into a 5 dBi antenna is ~+25 dBm EIRP — **illegal by ~11 dB**. Antenna gain counts. You will need to reduce PA output in firmware, which means the link budget must close at the *legal* power, not the silicon's capability.

## Related

- [[LoRa-Airtime-Budget]]
- [[Link-Budget-And-Site-Survey]]
- [[Open-Questions]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. NRFAT-2016 Rev. 2: https://region7.ntc.gov.ph/wp-content/uploads/2024/01/NRFAT-2016-Rev-2.pdf · RP002-1.0.3 Table 1 p.16 · NTC circulars index: https://ntc.gov.ph/memorandum-circulars/ · Type approval: https://www.csagroup.org/global-certification-regulatory-update/philippines-ntc-type-approval-acceptance-of-equipment-based-on-specific-radio-module/
