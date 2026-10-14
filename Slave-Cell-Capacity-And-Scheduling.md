# Slave Cell Capacity And Scheduling

How one slave gateway serves ~38–54 LoRa devices in its section without becoming a bottleneck. The short answer: **the MAC design matters more than the radio count.** Polling 54 devices doesn't work under a 1% duty cycle, whatever the number of radios. Scheduled uplinks do, and the second downstream radio then halves the cycle time.

Introduced 2026-10-14. Supersedes the "6 endpoint devices" assumption in [[LoRa-Airtime-Budget]] and [[Media-Access-Control-Options]].

## Cell sizes

Per-pole LoRa LED driver/monitoring devices plus one energy meter per section:

| Section | Lights | Devices on the slave's radio |
|---|---|---|
| 1 | 53 | ~54 |
| 2 | 38 | ~39 |
| 3 | 37 | ~38 |

One device per pole is confirmed (2026-10-14), but **the exact device count is not yet known**. The table uses the current light counts as a worked example. The design and the quote use a ceiling of **60 devices per section, 140 in total**: at 60 devices and SF9, scheduled uplink is ~17 s on one channel and ~8.5 s on two, still well inside a 60 s cycle.

*Earlier note:* ⚠ This assumes **one LED driver/monitoring device per pole**, inferred from the "~50 devices" annotation. If instead each device drives several poles, every number below gets easier. Tracked in [[Open-Questions]] Q3.

## Option 1: slave polls each device. Rejected.

16 B poll, 32 B reply, ~60 ms turnaround:

| SF | Per device | Sweep of 53 | Slave's TX per sweep | **Fastest legal sweep at 1% duty** |
|---|---|---|---|---|
| 7 | 183 ms | 9.7 s | 2.7 s | **~4.5 min** |
| 9 | 472 ms | 25 s | 8.7 s | **~15 min** |

The slave has to transmit once for every device, so **its own duty cycle sets the floor**, and adding radios doesn't fix that if the 1% limit is per box ([[Dual-Radio-Redundancy-And-Coexistence]]). This is exactly the bottleneck the client is worried about, and it comes from the protocol, not from the radio count.

## Option 2: scheduled uplink (cell TDMA). Recommended.

The slave transmits **one beacon per cycle** (time, slot map version, pending-command bitmap). Each device was given a fixed uplink slot at join and transmits its report in that slot, with no poll. The slave confirms everything received with **one group-ACK bitmap frame** per cycle.

32 B report (12 B header + 4 B MIC + 16 B data), 30 ms guard per slot:

| SF | Slot | 53 devices, **one channel** | **Two channels (both downstream radios receiving)** |
|---|---|---|---|
| 7 | 102 ms | 5.5 s | **2.8 s** |
| 8 | 164 ms | 8.9 s | **4.4 s** |
| 9 | 277 ms | 15.1 s | **7.5 s** |

- At a **60 s cycle and SF9**, one channel is 25% occupied and two channels 12.5%. Each device transmits 0.4% of the time, well under 1%.
- The slave only transmits a beacon plus a group ACK per cycle, plus any commands. **Its duty cycle stops being the limit.**
- If one downstream radio fails, everything moves to the surviving channel: 15 s per cycle at SF9, still inside 60 s.
- **Report-by-exception sits on top**: alarms go in a short contention window (slotted ALOHA + CAD) right after the beacon, instead of waiting for the device's slot ([[Media-Access-Control-Options]]).

**Recommended cadence:** 60 s device heartbeat/status. Energy meter aggregates every 5 min. Per-light detail on change, or every 15 min. Alarms immediately.

## Commands into the cell

- **Group and section commands are broadcast with `execute_at`.** The slave sends one frame on both channels, and every device switches at the same instant. This is how the [[Popcorn-Effect-And-Scheduled-Execution]] is avoided now that each light has its own radio node.
- **Confirmation comes from the next scheduled reports**, which carry `level_actual`. No per-device command ACK storm.
- **Unicast commands** (dim one pole) go in the beacon's pending bitmap. The device listens and ACKs in its slot.
- Devices are mains-powered, so they can **listen continuously** between their own transmissions ([[Continuous-Receive-Downlink]]). ⚠ On SX1276, continuous RX has the FIFO-pointer trap and needs careful driver handling.

## Timing coupling with tier 1

The slave's upstream radios can't transmit while its downstream radios are receiving ([[Dual-Radio-Redundancy-And-Coexistence]]). So the cell superframe reserves a **quiet window** that lines up with the master's tier-1 slot for that slave. Rough plan for a 60 s cycle:

```
0 s     cell beacon (both channels)
0.3 s   contention window (alarms), ~1 s
1.5 s   device uplink slots, ~8–15 s
~17 s   group ACK
~18 s   QUIET: tier-1 exchange with master (both directions), a few seconds
...     idle / retries / unicast commands / duty-cycle recovery
60 s    next beacon
```

**The master's slot grant for each slave must be designed together with this layout.** That is one of the two hardest design items in the system (the other is the cell join and slot assignment).

## Join and slot assignment

- A device joins via a `JOIN_REQ` in the contention window. The slave checks the device against the whitelist the master pushed to it, then assigns `{home_channel, slot_index, short_address}`.
- **The slot table is persisted on the slave** and mirrored to the master, so a slave reboot doesn't force 54 re-joins.
- After a mains restore, all 54 devices boot at once. **Joins must be randomised and back off**, or the contention window collapses. The fix: a device that already holds a valid slot assignment in its own non-volatile memory **resumes in its slot without re-joining**.

## Related

- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[LoRa-Airtime-Budget]]
- [[Media-Access-Control-Options]]
- [[Index]]

## Sources

- Client information via project owner, 2026-10-14: concern about "~50 devices connected at once" and a bottleneck, which motivated the second downstream radio.
- Airtime computed 2026-10-14 with the Semtech time-on-air formula ([[LoRa-Airtime-Budget]]): 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on.
