# Failure Modes And Mitigations

> **Updated 2026-10-14.** Dual radios change F1 and F2: a single radio, antenna or coax failure is now **non-service-affecting**, *if* the pair is truly independent (separate SPI bus, antenna, coax). F3/F4 (box death) are unchanged, since a second radio doesn't help when the MCU or PSU dies. New failure mode to add: **F16, in-box desensitisation**: one radio transmitting blinds the others in the same enclosure. Mitigate with a per-box TX scheduler and blanking windows. See [[Dual-Radio-Redundancy-And-Coexistence]]. **F17, mass re-join storm** after a mains restore with ~54 devices per cell. Mitigate with persisted slot assignments and randomised backoff ([[Slave-Cell-Capacity-And-Scheduling]]).

FMEA for the master gateway and its network. Severity is judged against a street-lighting context where **"dark road at night" is the worst outcome** — see [[Fail-Safe-Lighting-Rule]].

| # | Failure | Detection | Mitigation | Fail-safe outcome | Sev |
|---|---|---|---|---|---|
| F1 | Master↔slave link loss (fading, new building, antenna water ingress) | Missed beacons/heartbeats, both directions | Retries with backoff; SF escalation SF9→SF10→SF12; two antenna options at install; autonomous fallback | Section runs local schedule; alarm at server | High |
| F2 | Slave↔endpoint link loss | Slave-local heartbeat timeout | Same; keep last-known values as STALE; escalate as **one** alarm, not a storm | Driver device runs DALI fallback | High |
| F3 | **Slave gateway dies** (PSU, surge, lightning) | Master: node-missing after N superframes | Watchdog + supervisor; driver device holds its own astro fallback; spare unit on site | Section lights via driver fallback, else DALI System Failure Level | **Critical** |
| F4 | **Master gateway dies** | Server: Modbus timeouts. Slaves: no beacon | Hardware watchdog + brownout detect; redundant config flash; **pre-provisioned hot-spare master on a shelf**; slaves autonomous | Site runs on 3 autonomous schedules; no telemetry, lights work | **Critical** |
| F5 | Server dies / RS485 cut | Master: no Modbus poll for T | Store-and-forward ring buffer with original timestamps, replay on reconnect, FIFO overwrite + overflow alarm | No visibility; lighting unaffected | Medium |
| F6 | **Mains loss on one phase** | Meter reports zero V; slave last-gasp report (needs a supercap to *send* it); master infers "section dark" | Distinguish **commanded off** from **no power** — these must never share a status code | Fast, accurate alarming | **Critical (safety)** |
| F7 | **DALI bus fault** (short, open, water, gear dragging the bus) | Driver device: bus V/I out of range; missing replies from known addresses | Per-address retry; short detection; report affected address list; drivers set to `System Failure Level = 100%` | Lights go full brightness, uncontrolled — the correct fail-safe | **Critical** |
| F8 | Partial command delivery ("121 of 128 on") | Per-target ACK accounting | `PARTIAL` job state + fail mask; targeted retry of stragglers; **DALI broadcast, not 43 unicasts** | Reported honestly, retried | High |
| F9 | Clock drift | Master compares slave-reported time each uplink | Time in every beacon; TCXO on master, battery RTC on slaves. At 20 ppm drift is ~1.7 s/day — resync daily and you never exceed a few seconds | `CLOCK_UNTRUSTED` → lights ON | High |
| F10 | Congestion / duty-cycle exhaustion (usually self-inflicted retry storms) | Duty-cycle ledger, queue depth, retry counters | **Hard airtime reserve for P0**; exponential backoff; alarm rate-limiting and aggregation ("section has 12 lamp faults", not 12 frames); CAD before TX | Degraded telemetry, control preserved | High |
| F11 | Duplicate / replayed commands | Monotonic frame counter + replay window | Sequence numbers + MIC; sliding-window duplicate cache; **idempotent semantics** | Duplicates are no-ops | High |
| F12 | External interference / jamming | Noise-floor sampling in idle slots; PER trend; RSSI-without-valid-packet counter | Channel hopping/blacklisting; CAD; SF escalation; `RF_ENVIRONMENT_DEGRADED` alarm | Autonomous fallback | Med–High |
| F13 | Registry / config flash corruption | CRC on every record at boot | Dual-bank config + journal; factory-default recovery; **registry mirrored to the server and restorable** | Recoverable without re-commissioning | High |
| F14 | Firmware update bricks a slave | Version reporting; post-update heartbeat | A/B slots with rollback on failed self-test; **never update all 3 slaves at once**; daytime maintenance windows only | One section at a time, recoverable | High |
| F15 | Surge / lightning on RS485 or DALI | Comms dead, often permanently | Galvanic isolation on RS485 and DALI; TVS/GDT; single-point earthing. **Non-negotiable in the Philippines** — high lightning density plus poor cabinet earthing is the usual real cause of "unexplained comms failures" in tropical installs | Hardware survives | **Critical** |

## The test that matters

Every row above becomes a line in the **Gate 3 destructive test matrix** — see [[Development-Roadmap-And-Gates]]. For each injected fault: lights behave per the fail-safe rule, an alarm is raised, and the system self-recovers when the fault is removed, with no manual intervention and no invented data.

## Related

- [[Fail-Safe-Lighting-Rule]]
- [[Gateway-Observability]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09.
