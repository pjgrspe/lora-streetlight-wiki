# LoRa Airtime Budget

How much time on air each transmission costs, and what the Philippine radio limits allow. Bottom line: airtime and the duty-cycle limit, not collisions, constrain this network. The 400 ms dwell limit makes SF9 the cell design centre, and a 1% budget per box sets the **cycle at 120 s**, not 60 s.

## Key points

- Firmware implements the time-on-air formula as `lora_airtime_ms()`, host-tests it, and **gates every transmission on it**.
- Under the **400 ms dwell limit**, SF9 carries at most 66 B (54 B of frame payload) and SF7 255 B (243 B). **SF11/SF12 carry nothing.** v1 uses SF7–SF9 only.
- **The slave is the critical box.** At a 120 s cycle its scheduled traffic is **66%** of a 1% budget per box, with 25% reserved for P0.
- **A 60 s cycle doesn't fit** (~112%) unless NTC allows one budget per radio.
- Keep a **rolling 1-hour airtime ledger per box** (not per radio, not per message) with a **hard 25% reserve for P0**.

Specified in `Specs/specs/04-channel-plan-and-airtime.md` (v0.1).

## Time on air

```
Tsym      = 2^SF / BW
Tpreamble = (n_preamble + 4.25) × Tsym
n_payload = 8 + max( ceil((8·PL − 4·SF + 28 + 16·CRC − 20·IH) / (4·(SF − 2·DE))) × (CR+4), 0 )
Tpacket   = Tpreamble + n_payload × Tsym
```

At 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on (PHY size includes the 12-byte frame overhead):

| Frame | Size | SF7 | SF8 | SF9 |
|---|---|---|---|---|
| Tier-1 heartbeat | 16 B | 51 ms | 93 ms | 165 ms |
| Tier-1 beacon | 25 B | 62 ms | 113 ms | 206 ms |
| Tier-1 summary | 44 B | 92 ms | 164 ms | 288 ms |
| Tier-1 detail (20 records) | 252 B | 394 ms | — | — |
| Cell beacon | 27 B | — | — | 226 ms |
| LED device report | 32 B | 72 ms | 134 ms | 247 ms |
| Meter report | 56 B | 108 ms | 195 ms | 349 ms |

## The 400 ms dwell limit

| SF | Max PHY payload | Max frame payload (12 B overhead) |
|---|---|---|
| 7 | 255 B (399.6 ms) | 243 B |
| 8 | 138 B | 126 B |
| 9 | **66 B** | **54 B** ← cell design centre |
| 10 | 24 B | 12 B (not used in v1) |
| 11 / 12 | **0 B** (the minimum frame is already over 400 ms) | — |

Verified against LoRa Alliance RP002-1.0.3 Table 75.

> **There is no long-range fallback at SF11/SF12.** v1 goes further and stops at SF9 in cells: a link that doesn't close at SF9 needs antenna height or placement, not a slower data rate ([[Link-Budget-And-Site-Survey]]).

## Duty cycle: why the cycle is 120 s

Design limit: **≤1%, rolling 1 hour, per box** = 36 s of TX per hour per enclosure. A slave has 4 radios; until NTC says otherwise, they share one budget ([[Dual-Radio-Redundancy-And-Coexistence]]).

Polling a cell is hopeless under that limit (≥15 min per sweep of ~54 devices at SF9), so devices report in scheduled slots ([[Slave-Cell-Capacity-And-Scheduling]]). Even then, the slave's own scheduled traffic is significant, because it beacons on **both** cell channels every cycle and forwards detail upstream:

| Slave traffic at a 120 s cycle | Per hour |
|---|---|
| Cell beacons on 2 channels | 13.6 s |
| Tier-1 summary every cycle | 2.8 s |
| Standby heartbeat (every 5th cycle) | 0.3 s |
| Detail: basic records every 15 min, extended hourly, meter every 5 min | 7.2 s |
| **Scheduled total** | **23.9 s (66%)** |
| P0 reserve | 9.0 s (25%) |
| Left for commands, retries, joins | 3.1 s (9%) |

| Cycle | Scheduled slave load | Verdict |
|---|---|---|
| 60 s | ~112% | ❌ Doesn't fit a per-box budget |
| **120 s** | **66%** | ✅ Default |
| 180 s | ~51% | ✅ More margin, slower status |

- The margin for commands and retries is thin. The first lever is the detail interval: 30 min instead of 15 frees ~2.4 s/h.
- The master is not the limit: beacon + heartbeat is ~10% at 120 s.
- A device sends one report per cycle: 0.2% (LED) or 0.3% (meter).
- **If NTC confirms one budget per radio**, a 60 s cycle becomes possible. That question is in the NTC letter ([[Philippines-Regulatory-Context]]).

## Channel separation

On separate channels the tiers run concurrently, and each channel's occupancy stays 1×. That's why tier 1 and each section have their own channels: 2 for tier 1 plus 2 per section, from 14 channels on a 200 kHz raster. Up to 6 sections fit before pairs must be reused across distance. Details in [[Dual-Radio-Redundancy-And-Coexistence]] and spec 04.

| Link | Channels | SF |
|---|---|---|
| Tier 1: Master ↔ Slaves | ch 0 (primary) + ch 13 (standby), opposite band edges | SF7 (SF8 allowed) |
| Tier 2: each Slave ↔ its devices | ch n + ch n+6 for section n, both active | SF9 (SF7–SF9, one per cell) |

## Airtime accounting rules

- Keep a **rolling 1-hour ledger per box**: a relaying slave is both receiver and transmitter for the same logical frame, so per-message accounting double-counts or misses.
- Reserve **25% for P0**. Without it, a configuration push or a chatty diagnostic session could lock out an emergency ON command.
- Defer P1–P3 work when the budget reaches the reserve. The ledger is reported in diagnostics and the Modbus map.

## Related

- [[LoRa-PHY-Fundamentals]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Firmware-Update-Over-LoRa]]

## Sources

- `Specs/specs/04-channel-plan-and-airtime.md` (v0.1, 2026-10-21)
- Research synthesis, 2026-09-09. Semtech AN1200.13; LoRa Alliance RP002-1.0.3 Tables 70/75: https://lora-alliance.org/wp-content/uploads/2021/05/RP002-1.0.3-FINAL-1.pdf

## History

- 2026-09-09: Written for a ~9-node network (3 slaves + 6 endpoints); a full poll of 6 endpoints was 5.9 s at SF9.
- 2026-10-14: Network grew to per-pole devices (up to 60 per slave). Polling replaced by scheduled uplink, channel separation made mandatory, ledger made per box.
- 2026-10-14: Philippine radio profile adopted (915–918 MHz, ≤400 ms, ≤1%, ≤16 dBm EIRP).
- 2026-10-21: aligned with spec pack v0.1. 12-byte frame overhead; full slave budget computed (beacons on both channels plus detail forwarding); default cycle 120 s because 60 s needs ~112% of a per-box budget; v1 limited to SF7–SF9.

Back to [[Index]]
