# Slave Gateways As Intelligent Sub-Controllers

> **Updated 2026-10-14.** The slaves are now **pgorospe's deliverable** (no separate slave developer). Each slave has **4 SX1276 radios**: 2 upstream to the master, 2 downstream to the devices. Each serves **~38–54 devices** (per-pole LED driver/monitoring devices + 1 meter), not 2. The "polls its 2 endpoints locally" line below is superseded: polling 54 devices can't work under a 1% duty cycle, so the slave runs a **scheduled-uplink cell TDMA** instead. See [[Slave-Cell-Capacity-And-Scheduling]] and [[Dual-Radio-Redundancy-And-Coexistence]]. The autonomy argument in this note is **stronger** now: with 54 radio nodes per section, the nightly switch must not depend on the network.

The three per-phase slave gateways should be **intelligent sub-controllers** — each with its own scheduler, cache, RTC and astronomical clock — not dumb radio repeaters.

## The tradeoff

| Dimension | Dumb repeater | Intelligent sub-controller |
|---|---|---|
| Airtime | Every endpoint transaction crosses master↔slave↔device; the master's duty cycle throttles the whole site | Slave polls its 2 endpoints locally; only summaries and exceptions cross to the master — roughly 5–10x less traffic on the master link |
| Local loop latency | Two master round trips | Tens of ms, local |
| **Master link failure** | **Section goes dark or freezes** | Section runs its own astro schedule indefinitely |
| Firmware surface | Low | Three more things that can have bugs |
| Debuggability | One brain | Distributed state — "who has the right schedule?" |
| Clock hardware | Only master needs an RTC | Every slave needs RTC + backup + astro table |
| Cost | Marginally lower | +RTC, +flash, larger MCU: ~$5–15 x 3. Negligible. |

## Why the decision is not close

A star topology means a gateway failure darkens a whole neighbourhood. You cannot remove the master as a single point of failure, but you **can make its failure non-service-affecting** by pushing autonomy downward. Every credible commercial street-light controller does this: operation is autonomous from an astronomical calendar computed locally from stored lat/lon, with the network used for supervision and override rather than for the nightly on/off.

This is the same reasoning as [[Fail-Safe-Lighting-Rule]] — layered independence.

## Responsibility split

| Function | Master | Slave GW | Endpoint device |
|---|---|---|---|
| Authoritative UTC | **Owns** | Caches, drifts within tolerance | Caches |
| Astro schedule computation | Computes and distributes; validates | **Computes locally from lat/lon** (redundant on purpose) | Crude fallback |
| Nightly ON/OFF decision | Not in the loop | **Decides autonomously** | DALI executed by driver device |
| Polling meter / driver | No | **Yes** | Responds |
| DALI addressing | No | Requests | **Owns the DALI bus** |
| Aggregation, exception detection | Site-wide | **Section-wide** | Device-level |
| Command reception | From server | From master; **may refuse unsafe commands** | Executes |
| Event buffering | Site ring buffer | **Section ring buffer, 24 h+** | Small |
| Radio scheduling | Grants slots to slaves | Schedules its own 2 endpoints inside its slot | Obeys |

## Slave communication state machine

| State | Trigger | Behaviour |
|---|---|---|
| `SYNCED` | Beacon within 2 superframes | Follow master schedule and overrides; report normally |
| `DEGRADED` | No beacon 5–15 min | Keep buffering; keep local astro schedule; retain the last override **only if unexpired**; raise local alarm |
| `AUTONOMOUS` | No beacon > 1 h | **Discard all overrides**; run pure local astro schedule at 100%; keep buffering; rejoin with exponential backoff capped ~10 min |
| `CLOCK_UNTRUSTED` | RTC lost or drift beyond tolerance | **Lights ON continuously.** Wasted energy beats a dark highway. |
| `REJOINING` | Beacon returns | Re-sync clock, fetch config version, replay buffered events oldest-first with rate limiting, resume |

Two mandatory details:

1. **Overrides must expire.** A "dim to 30% for maintenance" command that outlives the master forever is how a street ends up permanently dim.
2. **The master must never silently overwrite an autonomous section on rejoin.** Compare `config_version_hash`, push only a delta, and log the divergence as an operator-visible event.

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Time-Distribution-And-Clock-Trust]]
- [[Failure-Modes-And-Mitigations]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Autonomous astro-calendar control as commercial norm: https://intelilight.eu/intelligent-street-lighting-control/ · single-point-of-failure critique: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/
