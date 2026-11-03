# Media Access Control Options

How the radios share the air: which access scheme (ALOHA, CSMA via CAD, TDMA) to use on each tier, and how traffic is prioritised. Bottom line: **master-clocked scheduled TDMA on both tiers**, a small contention window for alarms, and CAD only as a guard.

## Key points

- **Tier 1 (master ↔ slaves):** master-clocked **scheduled-uplink** superframe: a beacon, then each slave reports in its own 2 s slot. The master doesn't poll.
- **Tier 2 (slave ↔ up to 60 devices):** **scheduled-uplink cell TDMA across two channels.** Devices report in assigned slots, with no polling ([[Slave-Cell-Capacity-And-Scheduling]]).
- **Contention window** (slotted ALOHA + CAD) only for unsolicited alarms and joins.
- SX1276 **CAD is a guard, not the MAC**: preamble-only, and blind to hidden nodes.
- **Four priority classes** (P0–P3) with a hard 25% airtime reserve for P0.
- **One cycle for both tiers: 120 s** by default, set by the slave's duty-cycle budget ([[LoRa-Airtime-Budget]]).
- **Master-driven round-robin polling of endpoints is rejected.**

## The candidates

| Scheme | Max channel utilisation | Latency | Verdict |
|---|---|---|---|
| Pure ALOHA | **18.4%** | unbounded tail | ❌ for routine traffic |
| Slotted ALOHA | **36.8%** | bounded slot | ✅ for the alarm/join window only |
| CSMA via CAD | ~50–70%, impaired by hidden nodes | variable | A **guard**, not the MAC |
| **TDMA / scheduled slots** | **~90%+** | deterministic | ✅ **The primary MAC on both tiers** |

Pure ALOHA's 18.4% and slotted ALOHA's 36.8% are textbook figures, confirmed in the LoRa literature. LoRaWAN uplink behaves essentially as pure ALOHA and degrades fast under load. Slotted ALOHA over LoRaWAN measured a **5.8× throughput increase and 26% fewer collisions** at high load.

### Why ALOHA can't run a per-pole cell

130 nodes reporting autonomously under pure ALOHA at SF9, 20 B:

| Report period | Offered load G | Success probability |
|---|---|---|
| 30 s | 1.61 | **20%** ✗ |
| 60 s | 0.80 | **45%** ✗ |
| 300 s | 0.161 | 85% |
| 900 s | 0.054 | 95% |

ALOHA would need ≥5-minute reporting to be usable and ≥15 minutes to be reliable. **Scheduled TDMA across two channels serves a 60-device cell in under 10 s of airtime per cycle**, which is why the cell uses it. (The cycle is 120 s because of the slave's own duty budget, not cell capacity.) Even split per slave (~54 devices), ALOHA at 60–120 s is far below acceptable delivery.

## Channel Activity Detection on the SX1276: know its limits

CAD correlates received samples against the ideal preamble chirp, detecting LoRa **below the noise floor**, where RSSI-based carrier sense is useless.

- On the SX1276, CAD is a **fixed algorithm**: start CAD, then read `CadDone` / `CadDetected` (DIO0/DIO1). There are no tunable detection thresholds, and no automatic roll into receive on detection; the host must start RX itself after `CadDetected`.
- ⚠ Duration is roughly two symbols plus processing, about **8 ms at SF9**.
- **CAD detects the preamble only.** Running CAD in the middle of someone's payload returns "clear" and you collide. Mitigate with longer preambles or repeated CAD.
- **Hidden nodes:** measurements show CAD often fails to detect ongoing transmissions beyond ~400 m without line of sight. Along a street with devices spread across a section, hidden terminals are the *normal* case.
- **Inside a gateway box, CAD is irrelevant between its own radios.** The box's single TX scheduler already knows when its radios transmit ([[Dual-Radio-Redundancy-And-Coexistence]]).

## Recommended MAC

### Tier 1: master-clocked superframe

```
Master time (s), cycle = 120 s
0.0      BEACON on the active channel (ch 0): time, cycle number, config version,
         ACK bitmap for last cycle's slave summaries
0.5      HEARTBEAT on the standby channel (ch 13), every cycle
1.0–2.0  contention window: urgent slave events, slave joins
2 + 2(n−1)  slot of slave n, 2 s each, in GwID order:
         SUMMARY up → optional ACK/CMD down → optional DETAIL/EVENT/FRAG up → optional ACK down
end of slots → 120   master downlinks: commands, whitelist pushes, fragments, retries
```

Each slave's slot falls inside its cell's quiet window. Summaries are snapshots, so a missed one is simply replaced by the next. P0 commands don't wait for a window: the master sends them immediately on both radios and repeats them once.

### Tier 2: scheduled-uplink cell (per slave, both downstream channels)

```
cell time (s)
0.00 / 0.25  cell BEACON on channel A / channel B (time, versions, event-ACK bitmap)
0.60     contention window, 1 s: events and joins (slotted ALOHA + CAD)
1.60     uplink slots: 32 per channel, 0.3 s each → ends 11.2
12–16    QUIET window: the slave's tier-1 slot
16+      downlink window: commands, join accepts, fragments, retries
120      next cycle
```

Routine reports aren't acknowledged; events are, through the next beacon's bitmap. Details in [[Slave-Cell-Capacity-And-Scheduling]].

Rules:

- **Time sync comes free from the beacon**, timestamped at TxDone. Devices transmit 20 ms into their slot, ±20 ms ([[Time-Distribution-And-Clock-Trust]]).
- **The two superframes are designed together.** A slave can't receive from its devices while transmitting to the master, so each slave offsets its cell cycle to put its tier-1 slot in its quiet window.
- **CAD before contention-window transmissions.** It costs ~8 ms at SF9 and avoids an obvious collision. Inside an assigned slot the schedule already guarantees the channel.
- **Nothing transmits unsolicited outside a window, except P0.** Devices listen continuously, so downlink commands need no announcement.

## Priority queues

The same scheme runs in master, slaves and devices:

| Priority | Class | Behaviour |
|---|---|---|
| **P0** | Safety/control: "lights on now", group switching, section dark, mains loss | Pre-empts: sent immediately, not in a window; **on both tier-1 radios**, repeated once 1 s later; `execute_at` ≥ 5 s ahead; may use the P0 reserve |
| **P1** | Alarms and events | Queued ahead of telemetry; uses the contention window; kept until acknowledged |
| **P2** | Routine telemetry, per-luminaire detail, config sync | Best-effort, resumable, paced by the remaining airtime budget |
| **P3** | Bulk transfers (schedule tables, logs, future firmware) | Maintenance windows only; always yields; resumable |

## Recommended cadences

| Data class | Cadence | Mechanism |
|---|---|---|
| Device status report | every cycle (120 s) | Device's own slot (meter: double slot) |
| Section summary (state, counts, aggregate power, changed devices) | every cycle | **Computed at the slave**, sent in its tier-1 slot |
| Per-device detail to the master | basic every 15 min and on change; extended hourly | DETAIL frames in the slave's tier-1 slot |
| Meter data to the master | every 5 min | DETAIL meter record |
| Alarms | next contention window | EVENT, acknowledged by bitmap |

**Don't use master-driven or slave-driven round-robin polling of endpoints.** It doubles airtime, makes the polling gateway's duty cycle the bottleneck (≥15 min per sweep at SF9 for a 53-device cell), and dies badly under loss, since the gateway burns its budget retrying a dead node. Report-by-exception with a heartbeat cuts bandwidth 80–95% compared with continuous polling, while still confirming online/offline status.

## Related

- [[LoRa-Airtime-Budget]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Custom-Protocol-Frame-Design]]
- [[Popcorn-Effect-And-Scheduled-Execution]]

## Sources

- `Specs/specs/02-tier1-protocol.md`, `03-tier2-device-protocol.md`, `04-channel-plan-and-airtime.md` (v0.1, 2026-10-21)
- Research synthesis, 2026-09-09. https://arxiv.org/pdf/1705.05899 · https://arxiv.org/pdf/1902.09468 · https://arxiv.org/pdf/2002.10732 · Semtech AN1200.85: https://www.semtech.com/uploads/technology/LoRa/cad-ensuring-lora-packets.pdf · https://www.vtscada.com/report-by-exception-protocols-ensure-your-scada-wont-miss-a-beat/

## History

- 2026-09-09: Written for 3 slaves and 6 endpoints; CAD advice assumed SX1262 (parameterised CAD with `CAD_RX` exit).
- 2026-10-14: Per-pole devices (up to 60 per slave) made scheduled-uplink cell TDMA the tier-2 MAC; tier-1/cell superframes coupled by a quiet window; CAD reframed for SX1276; P0 duplicated on the backup radio.
- 2026-10-21: aligned with spec pack v0.1. Tier 1 is scheduled uplink too (no master polling); 120 s cycle on both tiers; no pending-command or group-ACK bitmaps; reports unacknowledged; CAD only before contention-window transmissions; cadences updated.

Back to [[Index]]
