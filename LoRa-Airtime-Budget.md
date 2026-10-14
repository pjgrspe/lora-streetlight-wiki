# LoRa Airtime Budget

> **Updated 2026-10-14.** The "~9 radio nodes" premise below is superseded. The network is now **up to ~140 devices (count TBC) + 3 slaves + master**, with dual radios at each gateway tier. The formulas and dwell table still hold. The "full poll of all 6 endpoint devices" figures do not. For a 54-device cell, polling needs a ≥15 min sweep at SF9 under 1% duty, while scheduled uplink takes 15 s on one channel or 7.5 s on two. See [[Slave-Cell-Capacity-And-Scheduling]]. The channel-separation advice below is now **mandatory**, and needs ≥8 channels ([[Dual-Radio-Redundancy-And-Coexistence]]). Keep the airtime ledger **per box**, not per radio, until NTC clarifies.

Airtime — not bandwidth, not collisions — is the binding constraint on this network. With only ~9 radio nodes, contention is a non-issue; **regulatory airtime limits and two-hop latency are what actually constrain the design.**

## Time on air

```
Tsym      = 2^SF / BW
Tpreamble = (n_preamble + 4.25) × Tsym
n_payload = 8 + max( ceil((8·PL − 4·SF + 28 + 16·CRC − 20·IH) / (4·(SF − 2·DE))) × (CR+4), 0 )
Tpacket   = Tpreamble + n_payload × Tsym
```

At 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on:

| SF | 10 B | 20 B | 51 B | 100 B |
|---|---|---|---|---|
| 7 | 41 ms | 57 ms | 103 ms | 174 ms |
| 8 | 72 ms | 103 ms | 185 ms | 308 ms |
| 9 | 144 ms | 185 ms | 329 ms | 554 ms |
| 10 | 289 ms | 371 ms | 616 ms | 1026 ms |
| 11 | 578 ms | 741 ms | 1315 ms | 2216 ms |
| 12 | 991 ms | 1319 ms | 2466 ms | 3940 ms |

**Implement this formula in firmware** as `lora_airtime_ms(...)`, unit-test it on the host, and gate every transmission on it. It is the guard that keeps you legal.

## The 400 ms dwell limit kills SF11 and SF12

If AS923's 400 ms dwell-time regime applies (see [[Philippines-Regulatory-Context]]):

| DR | SF | Max PHY payload under 400 ms |
|---|---|---|
| DR5 | 7 | 255 B (399.6 ms) |
| DR4 | 8 | 138 B |
| DR3 | 9 | **66 B** |
| DR2 | 10 | 24 B |
| DR1 | 11 | **0 bytes** — the minimum frame is already 414 ms |
| DR0 | 12 | **0 bytes** — 827 ms minimum |

Verified against LoRa Alliance RP002-1.0.3 Table 75 with exact agreement.

> **This is the single most important constraint in the project: under a 400 ms dwell limit, SF11 and SF12 at 125 kHz cannot legally carry any payload at all.** Your long-range fallback disappears, and the design centre becomes SF9 (50 B usable payload after headers), with SF7 for short elevated links.

## Transaction cost, two-hop

16 B request / 40 B response, sequential on one channel:

| SF | 1-hop transaction | **2-hop relayed** |
|---|---|---|
| 7 | 134 ms | **267 ms** |
| 8 | 247 ms | **494 ms** |
| 9 | 453 ms | **905 ms** |
| 10 | 864 ms | **1729 ms** |

Full poll of all 6 endpoint devices, 2-hop, with turnaround: **2.1 s at SF7, 5.9 s at SF9, 10.9 s at SF10.** A 10-second polling cycle at SF9 is comfortable.

## ⚠ The live compliance problem

At SF9 with a 10 s cycle, each **gateway radio** transmits ~450 ms per 10 s = **4.5% duty cycle**, which violates a 1% per-device limit. Three ways out:

1. Lengthen the cycle to ≥45 s.
2. **Split the two tiers onto different channels** — see below. This is the highest-leverage fix.
3. Confirm that a dwell-time rather than duty-cycle regime applies.

Unresolved until [[Philippines-Regulatory-Context]] is settled. Tracked in [[Open-Questions]].

## Break the multi-hop doubling with channel separation

A relayed frame costs 2x channel airtime **if both hops share a channel**. Give them separate channels and the tiers run concurrently — latency becomes the *sum* of the hops while each channel's occupancy stays 1x.

| Tier | Link | Channel | SF |
|---|---|---|---|
| 1 | master ↔ 3 slave gateways | e.g. 916.6 MHz | **SF7** (short, elevated, near-LOS) |
| 2 | slave gateway ↔ its devices | e.g. 916.8 MHz | **SF9** (street-level NLOS) |

915–918 MHz at 125 kHz leaves room for ~20 non-overlapping channels. Use them.

## Airtime accounting rules

- Keep a **rolling 1-hour airtime ledger per radio** (not per message — a relay is both receiver and transmitter for the same logical frame).
- Reserve a **hard airtime floor for P0 traffic** (say 25%). Without it, a firmware push or a chatty diagnostic session will lock out an emergency ON command.
- Refuse P2/P3 work when the budget drops below the reserve.

## Related

- [[LoRa-PHY-Fundamentals]]
- [[Media-Access-Control-Options]]
- [[Firmware-Update-Over-LoRa]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Semtech AN1200.13; LoRa Alliance RP002-1.0.3 Tables 70/75: https://lora-alliance.org/wp-content/uploads/2021/05/RP002-1.0.3-FINAL-1.pdf
