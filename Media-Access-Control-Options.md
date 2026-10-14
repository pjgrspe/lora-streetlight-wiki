# Media Access Control Options

> **Updated 2026-10-14.** "Only 3 slaves and 6 endpoints" no longer holds: each slave now has ~38–54 devices. The recommended superframe below survives at tier 1. At tier 2 it becomes a **scheduled-uplink cell TDMA across two channels** (beacon → contention window → device slots → group ACK → quiet window for tier 1). See [[Slave-Cell-Capacity-And-Scheduling]]. The "if per-luminaire LoRa is ever proposed" section below is **now the actual design case**; its conclusion (ALOHA can't do it, scheduled TDMA across channels can) is what the design follows.

## The candidates

| Scheme | Max channel utilisation | Latency | Verdict here |
|---|---|---|---|
| Pure ALOHA | **18.4%** | unbounded tail | Rare alarms only |
| Slotted ALOHA | **36.8%** | bounded slot | Good for the alarm window |
| CSMA via CAD | ~50–70%, impaired by hidden nodes | variable | A **guard**, not the MAC |
| **TDMA / scheduled polling** | **~90%+** | deterministic | ✅ **The primary MAC** |

Pure ALOHA's 18.4% and slotted ALOHA's 36.8% are textbook and confirmed in the LoRa literature. LoRaWAN uplink behaves essentially as pure ALOHA and degrades fast under load. Slotted ALOHA over LoRaWAN measured a **5.8x throughput increase and 26% fewer collisions** at high load.

**But with only 3 slaves and 6 endpoints you are nowhere near a capacity cliff.** Your constraint is airtime and latency, not collisions — so spend the headroom on *redundancy* (repeats, ack-with-state) rather than on more nodes.

## Channel Activity Detection — useful, but know its limits

CAD correlates received samples against the ideal preamble chirp, detecting LoRa **below the noise floor** where RSSI-based carrier sense is useless.

- Timing: Rx phase is `(2^SF + 32)/BW`. At SF9/125 kHz = 4.35 ms; total CAD ≈ **8 ms**.
- **SX126x is far better than SX127x here**: `SetCadParams(cadSymbolNum, cadDetPeak, cadDetMin, cadExitMode, cadTimeout)` with `cadExitMode = CAD_RX` rolls straight into reception on detection, with no host round trip. A common starting point is `cadDetPeak = SF + 13`, `cadDetMin = 10`, `cadSymbolNum = 8`.
- ⚠ Semtech's own guidance is blunt: choosing these values "is not easy and must be carefully tested". AN1200.48, which supposedly gives the table, was **not publicly retrievable** — expect to tune empirically.

**Two hard limits:**

1. **CAD detects preamble only.** CAD in the middle of someone's payload returns "clear" and you collide. Mitigate with longer preambles or repeated CAD.
2. **Hidden nodes.** Measurements show CAD frequently fails to detect ongoing transmissions in NLOS beyond ~400 m. In a street canyon with devices spread across 3 sections, hidden terminals are the *normal* case, not the exception.

## Recommended MAC: master-clocked polled TDMA with a contention window

```
Superframe (e.g. 10–60 s), master-clocked:
├─ Beacon slot        master → all GWs: time, superframe seq, config version, pending-job bitmap
├─ Poll slots × 3     master ↔ slave GW n: command + response          ← deterministic
├─ Cell slots × 3     each slave GW ↔ its 2 endpoints, sub-scheduled   ← deterministic
└─ Contention window  slotted ALOHA + CAD, unsolicited alarms only     ← 1–2 slots
                      (then idle, for duty-cycle recovery)
```

- **Time sync comes free from the beacon.** Nodes need only a few ms of slot alignment. Guard bands of ±20 ms absorb drift between beacons — see [[Time-Distribution-And-Clock-Trust]] for the resync-interval arithmetic.
- Frequency-offset tolerance is generous at SF ≤ 10 (±200 ppm), so a plain crystal suffices on devices; the gateways want a TCXO because their error is inherited by every link.
- **CAD before every transmission anyway**, even inside a scheduled slot. It costs ~8 ms at SF9 and catches a foreign system on your channel or your own schedule having drifted. Defence in depth, not the MAC.
- **Nothing transmits unsolicited except in the contention window.**

## Priority queues

Mirror this scheme in the slaves too:

| Priority | Class | Behaviour |
|---|---|---|
| **P0** | Safety/control: "lights on now", section-dark, mains loss | Pre-empts; may abort an in-flight routine transaction; guaranteed slot next beacon |
| **P1** | Alarms and events | Queued ahead of telemetry; interrupt window; retained until master-acked |
| **P2** | Routine telemetry, per-luminaire dumps, config sync | Best-effort, resumable, paced by the remaining airtime budget |
| **P3** | Firmware images | Maintenance windows only, always yields, resumable |

## Recommended cadences

| Data class | Cadence | Mechanism |
|---|---|---|
| Slave heartbeat + section summary | 60–120 s | Slave-initiated in its slot |
| Energy meter aggregates | 5 min | Slave-initiated |
| Luminaire summary per section (on/off/fault counts, aggregate W) | 5 min | **Computed at the slave** |
| Per-luminaire detail, all 128 | **15–30 min**, or on demand as a job | Fragmented, slave-paced |
| Alarms | **immediate, pre-emptive** | Slave interrupts the schedule |

**Do not use master-driven round-robin polling of endpoints.** It doubles airtime, makes the master's duty cycle the site bottleneck, and dies badly under loss (the master burns its budget retrying a dead node). Report-by-exception with heartbeat reduces bandwidth 80–95% versus continuous polling while still giving positive online/offline confirmation.

## If per-luminaire LoRa is ever proposed

Pre-computed, because this is the scope-creep scenario. 130 nodes autonomously reporting under pure ALOHA at SF9, 20 B:

| Report period | Offered load G | Success probability |
|---|---|---|
| 30 s | 1.61 | **20%** ✗ |
| 60 s | 0.80 | **45%** ✗ |
| 300 s | 0.161 | 85% |
| 900 s | 0.054 | 95% |

**130 ALOHA nodes on one channel need a ≥5-minute reporting period to be usable at all, and ≥15 minutes to be reliable.** Scheduled TDMA across multiple channels could do 60 s; ALOHA cannot. See [[LoRa-For-Control-Critique]].

## Related

- [[LoRa-Airtime-Budget]]
- [[Custom-Protocol-Frame-Design]]
- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://arxiv.org/pdf/1705.05899 · https://arxiv.org/pdf/1902.09468 · https://arxiv.org/pdf/2002.10732 · Semtech AN1200.85: https://www.semtech.com/uploads/technology/LoRa/cad-ensuring-lora-packets.pdf · https://www.vtscada.com/report-by-exception-protocols-ensure-your-scada-wont-miss-a-beat/
