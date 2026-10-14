# Slave Gateways As Intelligent Sub-Controllers

Each section's Slave Gateway is an **intelligent sub-controller** with its own scheduler, cache, RTC and astronomical clock, not a dumb repeater. That is what makes a master failure non-service-affecting.

## Key points

- One Slave per section; the section count is TBC (3 drawn, possibly 4+). All Slaves run the same firmware, delivered by pgorospe.
- **4 SX1276 radios per Slave:** 2 upstream to the master (primary + standby) and 2 downstream to the devices, both active on separate channels. One TX scheduler covers all 4 ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Each Slave serves up to **60 devices** (per-pole LED driver/monitoring devices + the section's energy meter). It runs a **scheduled-uplink cell TDMA**, not polling ([[Slave-Cell-Capacity-And-Scheduling]]).
- The nightly on/off is decided **locally**, from the Slave's own astronomical schedule. The network supervises and overrides; it is never needed to switch the lights.
- Overrides must expire, and the master must never silently overwrite an autonomous section on rejoin.

## Why the decision is not close

A star topology means a gateway failure darkens a whole neighbourhood. The master can't be removed as a single point of failure, but its failure can be made **non-service-affecting** by pushing autonomy downward. Every credible commercial street-light controller works this way: it runs from an astronomical calendar computed locally from stored lat/lon, and uses the network for supervision and override, not for the nightly switch. With dozens of radio nodes per section, this matters even more.

This is the same layered-independence reasoning as [[Fail-Safe-Lighting-Rule]].

| Dimension | Dumb repeater | Intelligent sub-controller |
|---|---|---|
| Airtime | Every device transaction crosses master↔slave↔device; the master's duty cycle throttles the site | Devices report locally in scheduled slots; only summaries and exceptions go upstream |
| Local loop latency | Two master round trips | Local |
| **Master link failure** | **Section goes dark or freezes** | Section runs its own astro schedule indefinitely |
| Firmware surface | Low | More code that can have bugs |
| Debuggability | One brain | Distributed state ("who has the right schedule?") |
| Clock hardware | Only the master needs an RTC | Every Slave needs RTC + backup + astro table |

## Responsibility split

| Function | Master | Slave | Device (per pole) |
|---|---|---|---|
| Authoritative UTC | **Owns** | Caches, drifts within tolerance | Caches |
| Astro schedule | Computes, distributes, validates | **Computes locally from lat/lon** (redundant on purpose) | Holds its own fallback (device developer) |
| Nightly ON/OFF decision | Not in the loop | **Decides autonomously** | Executes over DALI; falls back alone if the Slave is lost |
| Collecting device reports | No | **Yes**, in scheduled slots | Reports in its slot |
| Aggregation, exception detection | Site-wide | **Section-wide** | Device-level |
| Command reception | From server | From master; **may refuse unsafe commands** | Executes |
| Event buffering | Site ring buffer | **Section ring buffer, 24 h+** | Small |
| Radio scheduling | Grants tier-1 slots to Slaves | Runs its own cell schedule, aligned with its tier-1 slot | Obeys its assigned slot |

## Slave communication state machine

| State | Trigger | Behaviour |
|---|---|---|
| `SYNCED` | Beacon within 2 superframes | Follow master schedule and overrides; report normally |
| `DEGRADED` | No beacon for 5–15 min | Keep buffering; keep local astro schedule; keep the last override **only if unexpired**; raise local alarm |
| `AUTONOMOUS` | No beacon for > 1 h | **Discard all overrides**; run pure local astro schedule at 100%; keep buffering; rejoin with exponential backoff capped at ~10 min |
| `CLOCK_UNTRUSTED` | RTC lost or drift beyond tolerance | **Lights ON continuously.** Wasted energy beats a dark road |
| `REJOINING` | Beacon returns | Re-sync clock, fetch config version, replay buffered events oldest-first with rate limiting, resume |

Two mandatory details:

1. **Overrides must expire.** A "dim to 30% for maintenance" command that outlives the master is how a street ends up permanently dim.
2. **The master must never silently overwrite an autonomous section on rejoin.** Compare `config_version_hash`, push only a delta, and log the divergence as an operator-visible event.

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Time-Distribution-And-Clock-Trust]]
- [[Failure-Modes-And-Mitigations]]

## Sources

- Research synthesis, 2026-09-09. Autonomous astro-calendar control as commercial norm: https://intelilight.eu/intelligent-street-lighting-control/ · single-point-of-failure critique: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/

## History

- 2026-09-09: created assuming a separate slave developer, 1 radio and 2 endpoints per Slave, polled locally.
- 2026-10-14: Slave firmware moved to pgorospe; 4 radios per Slave; up to 60 per-pole devices served by cell TDMA instead of polling.

Back to [[Index]]
