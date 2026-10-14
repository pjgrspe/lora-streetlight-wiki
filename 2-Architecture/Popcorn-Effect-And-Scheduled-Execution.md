# Popcorn Effect And Scheduled Execution

The **popcorn effect** is lights in a group switching on one by one, sometimes over minutes, because commands go out as sequential unicasts. It's a documented, named failure in LoRaWAN street lighting and the most visible way a smart lighting system looks broken. The fix is **broadcast + "execute at time T"**, applied at both radio tiers.

## Key points

- Never send "turn on now" to a group. Send one frame carrying `{action, execute_at_utc = now + a few s}`.
- With a LoRa device on every pole, this is now mandatory, not an optimisation.
- Two-tier flow: the master multicasts to all Slaves (on both tier-1 radios); each Slave broadcasts the same `execute_at` into its cell on both channels; every device switches at T.
- Reliable time distribution is a prerequisite ([[Time-Distribution-And-Clock-Trust]]).
- Confirmation is slower than action: it arrives with the devices' next scheduled reports. Never gate the visible result on full confirmation.

## The fix: broadcast + execute_at

```
Server ─Modbus job─► Master
Master ─tier-1 multicast {action, execute_at = T}, sent on both radios─► every Slave
Each Slave ─cell broadcast {action, execute_at = T}, on both channels─► every device in its section
At T: every device drives its luminaire over DALI. All switch together.
```

Scheduled execution only works if every receiver agrees on the time, so time sync is a P1 requirement, not a nicety.

## Why this matters more now

The original design put only gateways and a couple of endpoints on the radio, with wired DALI for the last hop, so a DALI broadcast per section was enough. Now each pole has its own radio device ([[System-Architecture-As-Diagrammed]]). Sequential unicast to up to 60 devices per section would ripple visibly across the street. Broadcast with `execute_at` keeps it to one frame per tier.

## Latency targets

⚠ Targets, not measurements. Validate on the bench.

| Operation | Realistic target |
|---|---|
| Master accepts the job over Modbus | < 50 ms (cache/queue only) |
| Multicast reaches all Slaves | 1–3 s |
| Slaves broadcast into their cells | +1–2 s |
| **Perceived "all lights on"** | **a few seconds, synchronised (switch at T)** |
| Confirmation of every device | ~1–2 report cycles (the cell cycle is ~60 s) |
| Individual luminaire dim command | seconds, via the cell's pending-command bitmap |
| Alarm from a device visible at the server | seconds to tens of seconds (alarm contention window, then tier 1) |

Decouple action from confirmation in the UI.

## Related

- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Asynchronous-Command-Job-Pattern]]
- [[LoRa-For-Control-Critique]]

## Sources

- Research synthesis, 2026-09-09. Popcorn effect in LoRaWAN street lighting: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/ · synchronised execution via controlled delay: US10721700 · group multicast control: US12081358

## History

- 2026-09-09: written for ~9 radio nodes with a DALI broadcast per section.
- 2026-10-14: per-pole radio devices; execute_at broadcast now applied at both tiers, and latency targets restated.

Back to [[Index]]
