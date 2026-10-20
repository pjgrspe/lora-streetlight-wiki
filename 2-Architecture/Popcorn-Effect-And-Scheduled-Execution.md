# Popcorn Effect And Scheduled Execution

The **popcorn effect** is lights in a group switching on one by one, sometimes over minutes, because commands go out as sequential unicasts. It's a documented, named failure in LoRaWAN street lighting and the most visible way a smart lighting system looks broken. The fix is **broadcast + "execute at time T"**, applied at both radio tiers.

## Key points

- Never send "turn on now" to a group. Send one frame carrying `{action, execute_at, not_after}`, with `execute_at` at least 5 s ahead (default now + 8 s).
- With a LoRa device on every pole, this is now mandatory, not an optimisation.
- Two-tier flow: group commands are **P0**. The Master sends one broadcast on **both tier-1 radios** and repeats it 1 s later; each Slave broadcasts the same `execute_at` into its cell on both channels, also repeated; every device switches at T.
- Reliable time distribution is a prerequisite ([[Time-Distribution-And-Clock-Trust]]).
- Confirmation is slower than action: it arrives with the devices' next scheduled reports. Never gate the visible result on full confirmation.

## The fix: broadcast + execute_at

```
Server ─Modbus job─► Master
Master ─tier-1 broadcast {action, execute_at = T}, on both radios, repeated after 1 s─► every Slave
Each Slave ─cell broadcast {action, execute_at = T}, on both channels, repeated─► every device in its section
At T: every device drives its luminaire over DALI. All switch together.
```

Scheduled execution only works if every receiver agrees on the time, so time sync is a P1 requirement, not a nicety.

The copies on both radios and the repeat are byte-identical (same frame counter), so receivers execute the command once. A P0 broadcast may pre-empt a device's uplink slot; reports are snapshots, so the next cycle's report replaces a lost one.

## Why this matters more now

The original design put only gateways and a couple of endpoints on the radio, with wired DALI for the last hop, so a DALI broadcast per section was enough. Now each pole has its own radio device ([[System-Architecture-As-Diagrammed]]). Sequential unicast to up to 60 devices per section would ripple visibly across the street. Broadcast with `execute_at` keeps it to one frame per tier.

## Latency targets

Expected times from `Specs/specs/04-channel-plan-and-airtime.md` §8, at the default 120 s cycle. ⚠ Targets, not measurements. Validate on the bench.

| Operation | Realistic target |
|---|---|
| Master accepts the job over Modbus | < 50 ms (cache/queue only) |
| **Group on/off/dim, server command to lights switching** | **5–10 s, all lights together (switch at T)** |
| Single-pole command | up to ~30 s, in the cell's next downlink window (devices listen continuously, so no announcement is needed) |
| Confirmation of every device | ~2 report cycles (the cycle is 120 s) |
| Device status visible at the server | within one cycle (≤ 2 min) |
| Per-device power and energy at the server | within 15 min |
| Alarm from a device visible at the server | usually < 2 min (contention window, then the slave's tier-1 slot) |
| Device declared offline | after 3 missed reports (6 min) |

Decouple action from confirmation in the UI.

## Related

- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Asynchronous-Command-Job-Pattern]]
- [[LoRa-For-Control-Critique]]

## Sources

- `Specs/specs/02-tier1-protocol.md` §11.3–11.4, `Specs/specs/04-channel-plan-and-airtime.md` §6.4, §8 (v0.1)
- Research synthesis, 2026-09-09. Popcorn effect in LoRaWAN street lighting: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/ · synchronised execution via controlled delay: US10721700 · group multicast control: US12081358

## History

- 2026-09-09: written for ~9 radio nodes with a DALI broadcast per section.
- 2026-10-14: per-pole radio devices; execute_at broadcast now applied at both tiers, and latency targets restated.
- 2026-10-21: aligned with spec pack v0.1 (P0 path on both radios with a 1 s repeat, execute_at ≥ 5 s ahead, latencies at the 120 s cycle).

Back to [[Index]]
