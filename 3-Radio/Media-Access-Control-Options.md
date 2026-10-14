# Media Access Control Options

How the radios share the air: which access scheme (ALOHA, CSMA via CAD, TDMA) to use on each tier, and how traffic is prioritised. Bottom line: **master-clocked scheduled TDMA on both tiers**, a small contention window for alarms, and CAD only as a guard.

## Key points

- **Tier 1 (master ↔ slaves):** master-clocked polled TDMA superframe with a beacon.
- **Tier 2 (slave ↔ up to 60 devices):** **scheduled-uplink cell TDMA across two channels.** Devices report in assigned slots, with no polling ([[Slave-Cell-Capacity-And-Scheduling]]).
- **Contention window** (slotted ALOHA + CAD) only for unsolicited alarms and joins.
- SX1276 **CAD is a guard, not the MAC**: preamble-only, and blind to hidden nodes.
- **Four priority classes** (P0–P3) with a hard airtime reserve for P0.
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

ALOHA would need ≥5-minute reporting to be usable and ≥15 minutes to be reliable. **Scheduled TDMA across multiple channels achieves a 60 s cycle**, which is why the cell uses it. Even split per slave (~54 devices), ALOHA at 60 s is far below acceptable delivery.

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
Superframe (e.g. 60 s), master-clocked, on the primary tier-1 channel:
├─ Beacon               master → all slaves: time, superframe seq, config version, pending-job bitmap
│                       (heartbeat also sent on the backup radio's channel every superframe)
├─ Slave slots × N      master ↔ slave n: commands down, summaries/exceptions up  ← deterministic
│                       each slot falls inside slave n's cell QUIET window
└─ Contention window    slotted ALOHA + CAD: unsolicited slave alarms only
```

### Tier 2: scheduled-uplink cell (per slave, both downstream channels)

```
0 s     cell beacon on both channels (time, slot-map version, pending-command bitmap)
~0.3 s  contention window: alarms and joins (slotted ALOHA + CAD), ~1 s
~1.5 s  device uplink slots, devices split across the two channels
~17 s   group-ACK bitmap
~18 s   QUIET window: the slave's tier-1 exchange with the master
...     retries, unicast commands, duty-cycle recovery
60 s    next beacon
```

Rules:

- **Time sync comes free from the beacon.** Nodes need only a few ms of slot alignment; ±20 ms guards absorb drift between beacons ([[Time-Distribution-And-Clock-Trust]]).
- **The two superframes are designed together.** A slave can't receive from its devices while transmitting to the master, so its tier-1 slot must fall in its cell's quiet window.
- **CAD before every transmission anyway**, even inside a scheduled slot. It costs ~8 ms at SF9 and catches a foreign system on the channel, or drift in our own schedule.
- **Nothing transmits unsolicited except in a contention window.**

## Priority queues

The same scheme runs in master, slaves and devices:

| Priority | Class | Behaviour |
|---|---|---|
| **P0** | Safety/control: "lights on now", section dark, mains loss | Pre-empts; may abort an in-flight routine transaction; guaranteed slot at the next beacon; **sent on both tier-1 radios** |
| **P1** | Alarms and events | Queued ahead of telemetry; uses the contention window; kept until acknowledged |
| **P2** | Routine telemetry, per-luminaire detail, config sync | Best-effort, resumable, paced by the remaining airtime budget |
| **P3** | Bulk transfers (schedule tables, logs, future firmware) | Maintenance windows only; always yields; resumable |

## Recommended cadences

| Data class | Cadence | Mechanism |
|---|---|---|
| Device heartbeat + status | 60 s | Device's own scheduled slot |
| Energy meter aggregates | 5 min | Meter's scheduled slot |
| Section summary (on/off/fault counts, aggregate W) | 60–120 s up to the master | **Computed at the slave**, sent in its tier-1 slot |
| Per-luminaire detail | on change, or every 15 min | Fragmented, slave-paced |
| Alarms | **immediate, pre-emptive** | Contention window |

**Don't use master-driven or slave-driven round-robin polling of endpoints.** It doubles airtime, makes the polling gateway's duty cycle the bottleneck (≥15 min per sweep at SF9 for a 53-device cell), and dies badly under loss, since the gateway burns its budget retrying a dead node. Report-by-exception with a heartbeat cuts bandwidth 80–95% compared with continuous polling, while still confirming online/offline status.

## Related

- [[LoRa-Airtime-Budget]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Custom-Protocol-Frame-Design]]
- [[Popcorn-Effect-And-Scheduled-Execution]]

## Sources

- Research synthesis, 2026-09-09. https://arxiv.org/pdf/1705.05899 · https://arxiv.org/pdf/1902.09468 · https://arxiv.org/pdf/2002.10732 · Semtech AN1200.85: https://www.semtech.com/uploads/technology/LoRa/cad-ensuring-lora-packets.pdf · https://www.vtscada.com/report-by-exception-protocols-ensure-your-scada-wont-miss-a-beat/

## History

- 2026-09-09: Written for 3 slaves and 6 endpoints; CAD advice assumed SX1262 (parameterised CAD with `CAD_RX` exit).
- 2026-10-14: Per-pole devices (up to 60 per slave) made scheduled-uplink cell TDMA the tier-2 MAC; tier-1/cell superframes coupled by a quiet window; CAD reframed for SX1276; P0 duplicated on the backup radio.

Back to [[Index]]
