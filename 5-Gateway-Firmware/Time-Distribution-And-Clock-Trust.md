# Time Distribution And Clock Trust

How time flows from the server to the Master, Slaves and devices, and what happens when it can't be trusted. The bottom line: the Master is the time root but treats the server's time as a proposal; every node computes its own sunrise/sunset locally; and an untrusted clock means lights ON.

## Key points

- Time is a **P1 requirement**: it drives the astronomical schedule, scheduled execution (`execute_at`, against the [[Popcorn-Effect-And-Scheduled-Execution]]), TDMA slot alignment at both tiers, and every telemetry timestamp.
- **The Master sanity-checks the server's time** and rejects large jumps without an explicit force flag.
- **Timestamp beacons at the radio interrupt** (TxDone/RxDone on DIO0), not at enqueue.
- **Astronomical scheduling needs almost no accuracy** (±30 s is invisible), so a node without network time keeps the right schedule for weeks.
- **Clock untrusted → 100% ON** until time is re-established.

## The chain

```
NTP-synced PC ──▶ Local Monitoring Server
                    │ Modbus FC16 → unix_time(32) + tz   (a proposal)
                    ▼
              MASTER GATEWAY  (RTC + backup, calibrated)   ← time root
                    │ tier-1 beacon on both radios: {epoch, seq, sched_epoch, uncertainty}
                    ▼
              SLAVE GATEWAYS  (own RTC, own astro table)
                    │ tier-2 cell beacon on both channels
                    ▼
              devices (own fallback schedule)
```

## Don't trust the server's clock

If the Master simply adopts the server's time and that PC's clock is wrong, the whole site is mistimed with no independent check.

> **Give the Master a battery-backed, calibrated RTC and treat the server's time as a *proposal*.** Reject jumps beyond N minutes unless a force flag is set. Log every correction; **a steadily growing correction means a failing crystal.**

## Frequency references: crystals and TCXOs

- **Radio frequency reference.** A gateway's frequency error is inherited by every link it serves, and a Philippine cabinet interior reaches 60–80 °C. TCXOs are recommended for gateways. ⚠ **Common SX1276 modules (e.g. RFM95W) use a plain crystal.** Either choose a module variant with a TCXO, or accept a crystal and stay at 125 kHz bandwidth and SF ≤ 9, where offset tolerance is generous ([[Radio-Hardware-Selection]]).
- **RTC crystal (32.768 kHz LSE).** Load-capacitance mismatch is the main error source: a crystal specified for C_L = 7 pF fitted with 12.5 pF caps runs slow, with field reports of **+100 to +300 ppm**. Match C_L and allow 3–5 pF of stray.
- **Temperature dominates.** A tuning-fork crystal has a parabolic tempco of ~−0.035 ppm/°C² around 25 °C. In a 70 °C cabinet that's **≈ −71 ppm, about −6 s/day**, larger than the initial tolerance. Design to this number.
- **`RTC_CALR` smooth calibration** (available on STM32L4) corrects a *constant* offset of roughly ±480 ppm at ~0.95 ppm resolution. After calibration expect **<1 s/day**, plus periodic resync for the temperature drift.

## Backup power for the RTC

- **CR2032 + a low-leakage steering diode** is cleanest. VBAT draw is sub-µA, so a cell lasts years. Add 1 kΩ + 100 nF on VBAT to keep radio-band noise out of the backup domain.
- **Supercap** avoids battery replacement, but ⚠ **a fully discharged supercap looks like a short**, so charge it through a 100–470 Ω resistor. 0.22 F from 3.3 V to a 1.55 V floor at 1 µA lasts **~4.5 days**: enough for a maintenance outage, not a season.
- **Recommendation: CR2032 plus a 0.22 F supercap behind the charge resistor.**
- ⚠ **On the NUCLEO-L476RG, VBAT is tied to 3V3 by default.** Check the solder bridge in the board manual (UM1724) before fitting a backup cell. The final gateway board must bring VBAT out separately (gateway hardware requirements, [[STM32-Part-Selection]]).

## Resync interval: derive it from the guard band

`T ≤ guard / relative_drift`:

| Guard | Relative drift | Max resync interval |
|---|---|---|
| 20 ms | 40 ppm (two ±20 ppm crystals) | **500 s ≈ 8 min** |
| 100 ms | 40 ppm | 42 min |
| **100 ms** | **4 ppm (TCXO both ends)** | **7 hours** |

Tier-1 and tier-2 beacons go out every superframe anyway, so slot alignment is refreshed far more often than these limits. The Slave should still filter the offset (EWMA) *and* estimate its own rate error, so it can extrapolate between beacons and survive a few missed ones.

**Timestamp at the interrupt, not at enqueue.** Stamp the beacon at `TxDone` and its receipt at `RxDone` (DIO0 on SX1276), then subtract the known airtime. That removes the largest error term and gives millisecond-level sync.

## Astronomical scheduling needs almost no accuracy

Sunset at Manila's latitude (~14.6 °N) moves about **1 minute per day** at most, and the annual swing is only about an hour. **±30 s of clock error is invisible.**

So **compute sunrise/sunset on every node** from lat/long + date with the NOAA solar-position algorithm (~1 KB of code, no tables):

```
n     = days since 2000-01-01 12:00 UT
L     = 280.460 + 0.9856474·n
g     = 357.528 + 0.9856003·n
λ     = L + 1.915·sin g + 0.020·sin 2g
δ     = asin( sin(23.439°)·sin λ )
cos ω = (sin(−0.833°) − sin φ·sin δ) / (cos φ·cos δ)
sunset = 12:00 + ω/15° − EoT/60 − (λ_long − 15·TZ)/15°
```

(−0.833° = −0.583° refraction − 0.25° solar semidiameter.) At 14.6 °N / 121 °E / UTC+8: sunset ≈ 17:27 (early December) to 18:29 (mid-June). Add a configurable ±15 min offset, since many specifications switch at civil twilight (~20–40 lux) rather than geometric sunset.

> **This is the highest-value robustness decision in the system.** With lat/long in config, the lights keep the right schedule for years with the server absent. A month without any network time drifts ±3 min at worst, irrelevant for lighting. On the Blue Pill devices this needs soft-float, which is the device developer's code, not the protocol library.

## Clock failure is a lights-ON condition

If the RTC is invalid (dead backup plus a power cycle at 3 a.m.), the node doesn't know the time. **Default to 100% ON until time is re-established.** Never default to OFF on unknown time ([[Fail-Safe-Lighting-Rule]]).

## Related

- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Street-Lighting-Control-Logic]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]

## Sources

- Research synthesis, 2026-09-09. LSE load-capacitance field reports: https://community.st.com/t5/stm32-mcus-products/stm32f103-lse-load-capacitance-calculations-rtc/td-p/694548 · LoRa time-sync achievable accuracy: https://www.mdpi.com/1999-5903/18/2/80 · supercap inrush: https://mischianti.org/stm32-internal-rtc-clock-and-battery-backup-vbat/

## History

- 2026-09-09: written for the STM32WLE5 / SX1262 (TCXO driven from DIO3).
- 2026-10-14: updated for SX1276 modules (usually crystal-only), the NUCLEO-L476RG VBAT strap, beacons on both radios at both tiers, and the Blue Pill device fallback.

Back to [[Index]]
