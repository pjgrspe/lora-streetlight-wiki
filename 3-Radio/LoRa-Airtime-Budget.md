# LoRa Airtime Budget

How much time on air each transmission costs, and what the Philippine radio limits allow. Bottom line: airtime and regulatory limits, not collisions, constrain this network. The 400 ms dwell limit makes SF9 the design centre and rules out SF11/SF12.

## Key points

- Implement the time-on-air formula in firmware as `lora_airtime_ms()`, host-test it, and **gate every transmission on it**.
- Under the **400 ms dwell limit**, SF9 carries at most 66 B (50 B of app payload) and SF10 24 B (8 B). **SF11/SF12 carry nothing.**
- **Polling a ~54-device cell is not viable** under a 1% duty cycle (≥15 min per sweep at SF9). **Scheduled uplink** takes ~15 s on one channel, ~7.5 s on two ([[Slave-Cell-Capacity-And-Scheduling]]).
- **Separate channels per tier and per section are mandatory**: 2 tier-1 channels plus 2 per section ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Keep a **rolling 1-hour airtime ledger per box** (not per radio, not per message) with a **hard reserve for P0 traffic**.

## Time on air

```
Tsym      = 2^SF / BW
Tpreamble = (n_preamble + 4.25) × Tsym
n_payload = 8 + max( ceil((8·PL − 4·SF + 28 + 16·CRC − 20·IH) / (4·(SF − 2·DE))) × (CR+4), 0 )
Tpacket   = Tpreamble + n_payload × Tsym
```

At 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on:

| SF | 10 B | 20 B | 32 B | 51 B | 100 B |
|---|---|---|---|---|---|
| 7 | 41 ms | 57 ms | 72 ms | 103 ms | 174 ms |
| 8 | 72 ms | 103 ms | 134 ms | 185 ms | 308 ms |
| 9 | 144 ms | 185 ms | 247 ms | 329 ms | 554 ms |
| 10 | 289 ms | 371 ms | 453 ms | 616 ms | 1026 ms |
| 11 | 578 ms | 741 ms | — | 1315 ms | 2216 ms |
| 12 | 991 ms | 1319 ms | — | 2466 ms | 3940 ms |

## The 400 ms dwell limit

The project designs to a **≤400 ms per-transmission** limit ([[Philippines-Regulatory-Context]]). Maximum PHY payload per data rate:

| DR | SF | Max PHY payload | Usable app payload (16 B header + MIC) |
|---|---|---|---|
| DR5 | 7 | 255 B (399.6 ms) | 239 B |
| DR4 | 8 | 138 B | 122 B |
| DR3 | 9 | **66 B** | **50 B** ← design centre |
| DR2 | 10 | 24 B | 8 B ← tight |
| DR1 | 11 | **0 B** (the minimum frame is already 414 ms) | — |
| DR0 | 12 | **0 B** (827 ms minimum) | — |

Verified against LoRa Alliance RP002-1.0.3 Table 75, with exact agreement.

> **There is no long-range fallback at SF11/SF12.** Retry SF escalation stops at SF10. Links that don't close at SF10 need antenna height or placement fixes, not more SF ([[Link-Budget-And-Site-Survey]]).

## Duty cycle: why polling fails and scheduled uplink works

Design limit: **≤1% duty cycle, accounted per box.** A slave has 4 radios; until NTC says otherwise, they share one budget ([[Dual-Radio-Redundancy-And-Coexistence]]).

For a 53-device cell (16 B poll, 32 B reply):

| Approach | SF9 cycle | Slave TX per cycle | Verdict |
|---|---|---|---|
| Slave polls each device | 25 s per sweep | 8.7 s → **≥15 min sweep** at 1% | ❌ |
| Devices report in scheduled slots | 15 s (one channel) · 7.5 s (two channels) | One beacon + one group ACK | ✅ |

At the **60-device ceiling**, scheduled uplink at SF9 is ≈17 s on one channel and ≈8.5 s on two, inside a 60 s cycle. Each device transmits ~0.4% of the time. Full analysis: [[Slave-Cell-Capacity-And-Scheduling]].

## Channel separation

If both hops of a relayed frame share a channel, the frame costs **2×** channel airtime. On separate channels the tiers run concurrently: latency becomes the *sum* of the hops, while each channel's occupancy stays 1×.

| Link | Channels | SF |
|---|---|---|
| Tier 1: Master ↔ Slaves | 2 (primary + backup, at opposite edges of 915–918 MHz) | SF7–SF9 (short, elevated, near line of sight) |
| Tier 2: each Slave ↔ its devices | 2 per section, both active | SF9 (street-level, no line of sight) |

The 200 kHz raster across 915–918 MHz gives ~15 channels. That covers **2 + 2 × sections**, so about **6 sections** fit before sections must reuse channels, which then depends on physical separation.

## Airtime accounting rules

- Keep a **rolling 1-hour airtime ledger per box**: a relaying slave is both receiver and transmitter for the same logical frame, so per-message accounting double-counts or misses.
- Reserve a **hard airtime floor for P0 traffic** (e.g. 25%). Without it, a firmware push or a chatty diagnostic session will lock out an emergency ON command.
- Refuse P2/P3 work when the budget drops below the reserve.

## Related

- [[LoRa-PHY-Fundamentals]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Firmware-Update-Over-LoRa]]

## Sources

- Research synthesis, 2026-09-09. Semtech AN1200.13; LoRa Alliance RP002-1.0.3 Tables 70/75: https://lora-alliance.org/wp-content/uploads/2021/05/RP002-1.0.3-FINAL-1.pdf
- 32 B column and cell figures computed 2026-10-14 with the same formula.

## History

- 2026-09-09: Written for a ~9-node network (3 slaves + 6 endpoints); a full poll of 6 endpoints was 5.9 s at SF9.
- 2026-10-14: Network grew to per-pole devices (up to 60 per slave). Polling replaced by scheduled uplink, channel separation made mandatory, ledger made per box.
- 2026-10-14: Philippine radio profile adopted (915–918 MHz, ≤400 ms, ≤1%, ≤16 dBm EIRP).

Back to [[Index]]
