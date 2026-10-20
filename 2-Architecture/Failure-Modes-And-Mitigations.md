# Failure Modes And Mitigations

FMEA for the gateway network: what can fail, how it's detected, how it's mitigated, and how the lights behave. This is the engineering view. The contractual version, with exact timers, per-role behaviour and alarm names, is the fail-safe table in `Specs/specs/05-fail-safe-behaviour.md` (v0.1, rows F1–F19, numbered differently from this note). Severity is judged against a street-lighting context where **"dark road at night" is the worst outcome** ([[Fail-Safe-Lighting-Rule]]).

## Key points

- 17 failure modes. The critical ones are whole-box death (master or Slave), mains loss, DALI faults and surge.
- Dual radios make single radio, antenna or coax failures **non-service-affecting**, but only when each radio of a pair is independent (separate SPI bus, antenna and coax). They don't help when a whole box dies.
- Two failure modes come from the revised design: **F16 in-box desensitisation** and **F17 mass re-join storm**.
- Every row becomes a line in the Gate 3 destructive test matrix. The M6 acceptance tests T1–T5 in spec 05 (server down, master down, slave down, radio failed, clock untrusted) are the contracted subset.
- Alarm names follow the server alarm catalogue in `Specs/specs/07-server-data-model.md` §6.

## FMEA table

| # | Failure | Detection | Mitigation | Fail-safe outcome | Sev |
|---|---|---|---|---|---|
| F1 | Master↔Slave link loss (fading, new building, antenna water ingress) | 3 missed exchanges on the active radio; heartbeats on the standby; radio self-test | **Switch to the standby radio** (exercised every cycle, so its health is known); fail back after 5 clean cycles; no SF escalation in v1 (tier 1 runs SF7, SF8 at most); slave `DEGRADED` after 6 min, `AUTONOMOUS` after 60 min | Section runs its local schedule; `RADIO_FAILOVER` or `SECTION_LOST` at server | High |
| F2 | Slave↔device link loss | Slave-side missed reports in assigned slots | Device listens on the Slave's other channel after 2 missed beacons and enters `fallback_mode` after 3 missed cycles on both; offline after 3 missed reports; keep last-known values as STALE; escalate as **one** alarm, not a storm | Device runs its own fallback schedule | High |
| F3 | **Slave Gateway dies** (PSU, surge, lightning) | Master: no SUMMARY for 3 cycles (`SECTION_LOST`). Devices: no beacon on either channel | Watchdog + supervisor; each per-pole device holds its own astro fallback; spare unit on site. Dual radios don't help here | Section lights via device fallback (100% at night) within 6 min, else DALI System Failure Level | **Critical** |
| F4 | **Master Gateway dies** | Server: Modbus timeouts. Slaves: no beacon on either tier-1 radio | Hardware watchdog + brownout detect; redundant config storage; **pre-provisioned hot-spare master on a shelf**; Slaves autonomous | Site runs on local schedules; no telemetry, lights work | **Critical** |
| F5 | Server dies / RS485 cut | Master: no valid Modbus frame for 30 min (`SERVER_LOST`). Server: 5 failed polls (`MASTER_OFFLINE`) | Store-and-forward ring buffer with original timestamps, replayed on reconnect; FIFO overwrite + overflow alarm | No visibility; lighting unaffected | Medium |
| F6 | **Mains loss on one phase** | Meter reports phase loss / zero V; many devices on the phase go silent together; slave infers the pattern (`SECTION_POWER_LOST`) | Distinguish **commanded off** from **no power**; they must never share a status code | Fast, accurate alarming | **Critical (safety)** |
| F7 | **DALI fault** at a pole (short, open, water, gear dragging the bus) | Per-pole device: bus V/I out of range, missing replies | Retry; short detection; report the affected pole; drivers set to `System Failure Level` = 254 (100%) | That light goes to full brightness, uncontrolled: the correct fail-safe | **Critical** |
| F8 | Partial command delivery ("121 of 128 on") | Per-device confirmation in the next scheduled report | `PARTIAL` job state + fail mask; targeted retry of stragglers; **cell broadcast with execute_at, not dozens of unicasts** | Reported honestly, retried | High |
| F9 | Clock drift | Master compares Slave-reported time on each uplink | Time in every beacon; disciplined RTC on master, battery-backed RTC on Slaves. At 20 ppm drift is ~1.7 s/day, so a daily resync keeps it to seconds | `CLOCK_UNTRUSTED` → lights ON | High |
| F10 | Congestion / duty-cycle exhaustion (usually self-inflicted retry storms) | Airtime ledger (per box), queue depth, retry counters | **Hard airtime reserve for P0**; exponential backoff; alarm rate-limiting and aggregation ("section has 12 lamp faults", not 12 frames); CAD before TX | Degraded telemetry, control preserved | High |
| F11 | Duplicate / replayed commands | Monotonic frame counter + replay window | Sequence numbers + MIC; sliding-window duplicate cache; **idempotent semantics**; duplicate suppression across the two radios of a pair | Duplicates are no-ops | High |
| F12 | External interference / jamming | Noise-floor sampling in idle slots; PER trend; RSSI-without-valid-packet counter | Frequency diversity across the paired radios' channels; failover to the standby channel (tier 1) or single-channel mode (cell); CAD; `RF_DEGRADED` alarm | Autonomous fallback | Med–High |
| F13 | Registry / config storage corruption | CRC on every record at boot | Dual-copy config + journal; factory-default recovery; **registry mirrored to the server and restorable** | Recoverable without re-commissioning | High |
| F14 | Firmware update bricks a Slave | Version reporting; post-update heartbeat | A/B images with rollback on failed self-test; **never update all Slaves at once**; daytime maintenance windows only. (OTA is not in the current engagement) | One section at a time, recoverable | High |
| F15 | Surge / lightning on RS485 or DALI | Comms dead, often permanently | Galvanic isolation on RS485 and DALI; TVS/GDT; single-point earthing. **Non-negotiable in the Philippines**: high lightning density plus poor cabinet earthing is the usual real cause of "unexplained comms failures" in tropical installs | Hardware survives | **Critical** |
| F16 | **In-box desensitisation**: one radio transmitting blinds the other radios in the same enclosure | RX errors or missed frames correlated with own-box TX; measured antenna isolation ([[Dual-Radio-Redundancy-And-Coexistence]]) | **One TX scheduler per box**; blanking windows during which other radios' RX is ignored and not counted as link loss; tier-1 slots aligned with each cell's quiet window | No false link-loss alarms; traffic scheduled around own TX | High |
| F17 | **Mass re-join storm** after a mains restore (up to 60 devices per cell boot at once) | Contention-window collision rate; join backlog | Devices with a stored slot **rejoin in their own slot** (`REJOIN_REQ`), never in the contention window; randomised backoff only for genuine first joins; Slave persists the device and slot table in FRAM ([[Slave-Cell-Capacity-And-Scheduling]]) | Cell recovers within a few cycles; devices run local schedules meanwhile | High |

## The test that matters

Every row becomes a line in the **Gate 3 destructive test matrix** ([[Development-Roadmap-And-Gates]]). For each injected fault: lights behave per the fail-safe rule, an alarm is raised, and the system self-recovers when the fault is removed, with no manual intervention and no invented data. For the dual-radio rows, the test is physical: disconnect one antenna and confirm failover.

## Related

- [[Fail-Safe-Lighting-Rule]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Gateway-Observability]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]

## Sources

- `Specs/specs/05-fail-safe-behaviour.md`, `Specs/specs/07-server-data-model.md` §6 (v0.1)
- Research synthesis, 2026-09-09.

## History

- 2026-10-14: dual radios folded into F1/F2/F12; device fallback moved to the per-pole devices; SF escalation capped at SF10 under the decided 400 ms dwell; F16 and F17 added.
- 2026-10-21: aligned with spec pack v0.1 (no SF10 escalation in v1; spec timers; alarm names from 07; rejoin in own slot; link to the contractual table in spec 05).

Back to [[Index]]
