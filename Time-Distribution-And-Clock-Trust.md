# Time Distribution And Clock Trust

Time is a P1 requirement, not a nicety: it underpins the astronomical schedule, scheduled execution (which defeats the [[Popcorn-Effect-And-Scheduled-Execution]]), TDMA slot alignment, and every timestamp in the telemetry cache.

## The chain

```
NTP / GPS ──▶ Local Monitoring Server
                │ Modbus FC16 → unix_time(32) + tz
                ▼
          MASTER GATEWAY  (RTC, TCXO, calibrated)   ← time root
                │ LoRa beacon: {epoch, seq, sched_epoch, uncertainty}
                ▼
          3 × SLAVE GATEWAYS  (own RTC, own astro table)
                │
                ▼
          endpoint devices
```

## Don't trust the server's clock

If the master takes UTC from the server over Modbus and the server's clock is wrong — very common on an un-NTP'd local Windows box — the whole site is mistimed with no independent check.

> **Give the master a battery-backed TCXO RTC (or a GPS module) and treat the server's time as a *proposal* to be sanity-checked.** Reject jumps beyond N minutes without an explicit force flag. Cheap insurance against a whole class of embarrassing incidents.

Log every correction. **A steadily growing correction means a failing crystal.**

## Crystal reality

- Load-capacitance mismatch is the dominant error source. A crystal specced for C_L = 7 pF driven with 12.5 pF caps runs slow — field reports of **+100 to +300 ppm** from exactly this mistake. Match C_L, allow 3–5 pF of stray, check drive level against ESR.
- A correctly loaded ±20 ppm crystal gives **1.7 s/day**.
- **Temperature dominates:** a 32 kHz tuning-fork crystal has a parabolic tempco of ~−0.035 ppm/°C² about a ~25 °C turnover. In a 70 °C cabinet that is **−71 ppm ≈ −6 s/day** — larger than the initial tolerance, and the number you must actually design to.
- **`RTC_CALR` smooth calibration** corrects roughly −487 to +489 ppm at ~0.95 ppm resolution by adding/masking LSE pulses over 32 s. After calibration you hold **<1 s/day**. It corrects a *constant* offset, not the temperature curve — so combine it with periodic resync.

## Backup power

- **CR2032 + a low-leakage steering diode** is cleanest; VBAT draw is sub-µA so a 220 mAh cell lasts years. Add 1 kΩ + 100 nF on VBAT to keep radio-band noise off the backup domain.
- **Supercap** avoids battery replacement, but ⚠ **a fully discharged supercap looks like a short and will draw large inrush** — charge it through a 100–470 Ω series resistor. Sizing: 0.22 F from 3.3 V to a 1.55 V floor at 1 µA ≈ **4.5 days**. Enough for a maintenance outage, not a season.
- **Recommendation: CR2032 plus a 0.22 F supercap behind the charge resistor.** ~$1.
- ⚠ **Verify VBAT is actually bonded out** in your chosen package or module — some modules tie VBAT to VDD internally, which makes backup impossible and forces an external RTC. Tracked in [[Open-Questions]].

## Resync interval — derive it from the guard band

`T ≤ guard / relative_drift`:

| Guard | Relative drift | Max resync interval |
|---|---|---|
| 20 ms | 40 ppm (two ±20 ppm crystals) | **500 s ≈ 8 min** |
| 100 ms | 40 ppm | 42 min |
| **100 ms** | **4 ppm (TCXO both ends)** | **7 hours** |

**Recommendation: TCXO on the gateways, 100 ms guard, beacon every 15 minutes.** Have the slave apply EWMA filtering to the offset *and* estimate its local rate error so it can extrapolate between beacons — the standard industrial pattern.

**Timestamp at the interrupt, not at enqueue:** stamp the beacon at `TxDone` and the receipt at `RxDone`, then subtract the known airtime. This removes the largest error term and gets you sub-millisecond to few-millisecond sync.

## Astronomical scheduling needs almost no accuracy

Sunset in Manila (~14.6 °N) moves about **1 minute per day** at the solstices, and the annual swing is only ~1 hour because the site is near the tropics. **±30 s of clock error is invisible.**

So: **compute sunrise/sunset on the gateway** from lat/long + date using the NOAA solar-position algorithm (~1 KB of code, no tables) rather than downloading a schedule.

```
n     = days since 2000-01-01 12:00 UT
L     = 280.460 + 0.9856474·n
g     = 357.528 + 0.9856003·n
λ     = L + 1.915·sin g + 0.020·sin 2g
δ     = asin( sin(23.439°)·sin λ )
cos ω = (sin(−0.833°) − sin φ·sin δ) / (cos φ·cos δ)
sunset = 12:00 + ω/15° − EoT/60 − (λ_long − 15·TZ)/15°
```

(−0.833° = −0.583° refraction − 0.25° solar semidiameter.) For 14.6 °N / 121 °E / UTC+8: sunset ≈ 17:27 (early Dec) to 18:29 (mid-Jun); night length 11 h 05 m to 12 h 55 m. Add a configurable ±15 min offset, since many specs switch at ~20–40 lux, i.e. civil twilight rather than geometric sunset.

> **This is the single highest-value robustness decision in the system:** with lat/long in config, the lights keep the right schedule for *years* with the server absent. A total comms blackout degrades gracefully — ±6 s/day worst case × 30 days = ±3 min, irrelevant for lighting.

## Clock failure is a lights-ON condition

If the RTC is invalid (dead backup cell plus a power cycle at 3 a.m.) you do not know the time. **Default to 100% ON until time is re-established.** Never default to OFF on unknown time — see [[Fail-Safe-Lighting-Rule]].

## Related

- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Street-Lighting-Control-Logic]]
- [[Media-Access-Control-Options]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. LSE load-capacitance field reports: https://community.st.com/t5/stm32-mcus-products/stm32f103-lse-load-capacitance-calculations-rtc/td-p/694548 · LoRa time-sync achievable accuracy: https://www.mdpi.com/1999-5903/18/2/80 · supercap inrush: https://mischianti.org/stm32-internal-rtc-clock-and-battery-backup-vbat/
