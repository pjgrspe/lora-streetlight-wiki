# Popcorn Effect And Scheduled Execution

> **Updated 2026-10-14.** Each light now appears to have its own radio device, so the "9 radio nodes / DALI broadcast per section" argument below no longer holds. **The fix still works, applied at both tiers:** the master multicasts `{action, execute_at}` to the 3 slaves (on both tier-1 radios), and each slave **broadcasts the same `execute_at` frame into its cell on both channels**. Every device switches at T, so it stays synchronised. Confirmation then comes from the devices' next scheduled reports, ~60 s later. See [[Slave-Cell-Capacity-And-Scheduling]].

The **popcorn effect** is the asynchronous activation of devices in a logically grouped set, caused by sequential unicast transmission and cumulative latency. On LoRaWAN street lighting deployments it is a documented, named problem: lights activate sporadically over minutes — sometimes hours — rather than simultaneously. It is the most visible way a smart lighting system looks broken to the public.

## The fix: multicast + "execute at time T"

Do not send "turn on now." Send **one multicast frame** carrying:

```
{ action, execute_at_utc = now + 3 s }
```

All three slave gateways receive it and fire their DALI broadcast at the same instant. The result is a synchronised switch-on with a single radio transmission.

This is why [[Time-Distribution-And-Clock-Trust]] is a P1 requirement and not a nicety — scheduled execution is worthless if the receivers disagree about what time it is.

## Why this architecture makes it easy

Our topology puts only **9 radio nodes** on the air (3 slaves + 6 endpoints); the last hop to 128 luminaires is **wired DALI**, which is fast and deterministic. So:

- One master transmission reaches all three sections.
- Each section does a **DALI broadcast**, not 43 unicasts.
- Perceived "all lights on" is achievable in **2–5 s, synchronised**.

Had LoRa gone to every luminaire, the popcorn effect would be close to unavoidable. **This is the architectural line to defend against scope creep** — see [[LoRa-For-Control-Critique]].

## Latency targets

| Operation | Realistic target (SF9, 2-hop) |
|---|---|
| Master accepts job over Modbus | < 50 ms (cache/queue only) |
| Multicast group ON reaches 3 slaves | 1–3 s |
| Slaves drive DALI broadcast | +0.2–0.5 s |
| **Perceived "all lights on"** | **2–5 s, synchronised** |
| Confirmation of all 128 individually | 30–120 s |
| Individual luminaire dim command | 1.5–4 s |
| Alarm from luminaire visible at server | 2–15 s |

Note the split: **confirmation is inherently slower than action.** Decouple them in the UI, and never gate the visible result on full confirmation.

## Related

- [[Asynchronous-Command-Job-Pattern]]
- [[LoRa-Airtime-Budget]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Popcorn effect in LoRaWAN street lighting: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/ · synchronised execution via controlled delay: US10721700 · group multicast control: US12081358
