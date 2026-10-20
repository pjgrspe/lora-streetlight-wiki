# Slave Gateways As Intelligent Sub-Controllers

Each section's Slave Gateway is an **intelligent sub-controller** with its own scheduler, cache, RTC and astronomical clock, not a dumb repeater. That is what makes a master failure non-service-affecting.

## Key points

- One Slave per section. The number of sections isn't fixed (configured 1–14). All Slaves run the same firmware, delivered by pgorospe.
- **4 SX1276 radios per Slave:** 2 upstream to the master (primary + standby) and 2 downstream to the devices, both active on separate channels. One TX scheduler covers all 4 ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Each Slave serves up to **60 devices** (per-pole LED driver/monitoring devices + the section's energy meter). It runs a **scheduled-uplink cell TDMA**, not polling ([[Slave-Cell-Capacity-And-Scheduling]]).
- The nightly on/off is decided **locally**, from the Slave's own astronomical schedule. The network supervises and overrides; it is never needed to switch the lights.
- `DEGRADED` after 6 min without the Master, `AUTONOMOUS` after 60 min. Overrides always expire (max 720 min) and are dropped on `AUTONOMOUS`.
- The Master must never silently overwrite an autonomous section on rejoin.

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

Specified in `Specs/specs/02-tier1-protocol.md` §14 (v0.1). Timers count cycles (default 120 s), so they scale if the cycle changes.

| State | Trigger | Behaviour |
|---|---|---|
| `SYNCED` | Tier-1 beacon within the last 2 cycles | Follow the Master's configuration and overrides; report normally |
| `DEGRADED` | **3 missed beacons (6 min)** | Keep the local astro schedule; keep an override **only while it is unexpired**; buffer events; raise an event |
| `AUTONOMOUS` | **30 missed beacons (60 min)** | **Discard all overrides**; run the pure local astro schedule (100% at night, no dimming); flag it in the cell beacon; keep buffering; try to rejoin with backoff capped at 5 cycles (10 min) |
| `CLOCK_UNTRUSTED` | Clock not trusted, in any state | **Lights 100% (level 254)** until time is restored. Overrides ignored. Wasted energy beats a dark road |
| `REJOINING` | Beacon heard again, or after boot | Rejoin, resync time, send the config hash; replay buffered events oldest first with their original timestamps |

Mandatory details:

1. **Overrides must expire.** Every override has a lifetime of at most 720 min, and all overrides are cleared on `AUTONOMOUS` and on reboot. A "dim to 30% for maintenance" command that outlives the Master is how a street ends up permanently dim.
2. **The Master must never silently overwrite an autonomous section on rejoin.** It compares the slave's `config_hash`, pushes only the changed items, and logs the divergence as an operator-visible event (`CONFIG_DIVERGED`). Overrides from before the outage are not re-applied.
3. **A slave may refuse an unsafe command**, e.g. dimming below the schedule while its clock is untrusted.
4. **Upstream radio failover** is per slave: after 3 missed beacons on the active channel (while still hearing the Master's heartbeat on the other), it moves to its standby radio; it fails back after 5 clean cycles (10 min).

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Time-Distribution-And-Clock-Trust]]
- [[Failure-Modes-And-Mitigations]]

## Sources

- `Specs/specs/02-tier1-protocol.md` §12, §14; `Specs/specs/05-fail-safe-behaviour.md` (v0.1)
- Research synthesis, 2026-09-09. Autonomous astro-calendar control as commercial norm: https://intelilight.eu/intelligent-street-lighting-control/ · single-point-of-failure critique: https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/

## History

- 2026-09-09: created assuming a separate slave developer, 1 radio and 2 endpoints per Slave, polled locally.
- 2026-10-14: Slave firmware moved to pgorospe; 4 radios per Slave; up to 60 per-pole devices served by cell TDMA instead of polling.
- 2026-10-21: aligned with spec pack v0.1 (state timers in cycles: DEGRADED 3, AUTONOMOUS 30; override max 720 min; upstream failover rules).

Back to [[Index]]
