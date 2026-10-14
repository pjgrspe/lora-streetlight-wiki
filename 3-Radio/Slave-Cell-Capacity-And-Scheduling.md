# Slave Cell Capacity And Scheduling

How one Slave Gateway serves all the LoRa devices in its section (one per pole, up to 60) without becoming a bottleneck. Bottom line: **the MAC design matters more than the radio count.** Polling can't work under a 1% duty cycle. Scheduled uplink does, and the second downstream radio then halves the cycle time.

## Key points

- **Design ceiling: 60 devices per slave.** One device per pole is confirmed; the exact count per section is not yet known.
- **Polling is rejected:** a 53-device sweep at SF9 needs **≥15 minutes** under 1% duty, because the slave must transmit once per device.
- **Scheduled uplink (cell TDMA):** one beacon, devices report in assigned slots, one group-ACK bitmap. At SF9: **≈15 s (53 devices) / ≈17 s (60) on one channel, ≈7.5 s / ≈8.5 s on two**, inside a 60 s cycle.
- If one downstream radio fails, the whole cell fits on the surviving channel.
- Group commands are **broadcast with `execute_at`** on both channels; confirmation comes from the next reports.
- The cell superframe reserves a **quiet window** for the slave's tier-1 exchange with the master.

## Cell sizes

| Section (as drawn) | Lights | Devices on the slave's radios |
|---|---|---|
| 1 | 53 | ~54 (53 LED devices + 1 meter) |
| 2 | 38 | ~39 |
| 3 | 37 | ~38 |
| **Any section (design ceiling)** | — | **60** |

The diagram shows three sections, but there may be four or more. The firmware and analysis are per cell and don't depend on the number of sections. The tables below use 53 as a worked example and 60 as the ceiling.

## Option 1: the slave polls each device (rejected)

16 B poll, 32 B reply, ~60 ms turnaround, 53 devices:

| SF | Per device | Sweep | Slave TX per sweep | **Fastest legal sweep at 1% duty** |
|---|---|---|---|---|
| 7 | 183 ms | 9.7 s | 2.7 s | **~4.5 min** |
| 9 | 472 ms | 25 s | 8.7 s | **~15 min** |

The slave transmits once for every device, so **its own duty cycle sets the floor**. Extra radios don't help, because the 1% budget is accounted per box ([[Dual-Radio-Redundancy-And-Coexistence]]). This is exactly the bottleneck the client was worried about, and it comes from the protocol, not the radio count.

## Option 2: scheduled uplink (cell TDMA), recommended

The slave transmits **one beacon per cycle** (time, slot-map version, pending-command bitmap). Each device has a fixed uplink slot assigned at join and transmits its report there, with no poll. The slave confirms everything received with **one group-ACK bitmap frame** per cycle.

32 B report (12 B header + 4 B MIC + 16 B data), 30 ms guard per slot:

| SF | Slot | 53 devices, one channel | 53 devices, **two channels** | 60 devices, one channel | 60 devices, **two channels** |
|---|---|---|---|---|---|
| 7 | 102 ms | 5.5 s | 2.8 s | 6.2 s | 3.1 s |
| 8 | 164 ms | 8.9 s | 4.4 s | 10.0 s | 5.0 s |
| 9 | 277 ms | 15.1 s | **7.5 s** | 17.1 s | **8.5 s** |

- At a **60 s cycle and SF9**, 60 devices occupy ~28% of one channel, or ~14% of each of two. Each device transmits ~0.4% of the time, well under 1%.
- The slave only sends a beacon plus a group ACK each cycle, plus any commands, so **its duty cycle stops being the limit**.
- **If one downstream radio fails**, everything moves to the surviving channel: ~17 s per cycle at SF9 for 60 devices, still inside 60 s.
- **Report-by-exception sits on top:** alarms go in a short contention window (slotted ALOHA + CAD) right after the beacon, rather than waiting for the device's slot ([[Media-Access-Control-Options]]).

**Cadence:** device heartbeat/status every 60 s; energy meter aggregates every 5 min; per-light detail on change or every 15 min; alarms immediately.

## Commands into the cell

- **Group and section commands are broadcast with `execute_at`.** The slave sends one frame on both channels, and every device switches at the same instant. This avoids the [[Popcorn-Effect-And-Scheduled-Execution]] even though each light has its own radio.
- **Confirmation comes from the next scheduled reports**, which carry `level_actual`. No per-device command-ACK storm.
- **Unicast commands** (dim one pole) are flagged in the beacon's pending-command bitmap. The device listens and ACKs in its slot.
- Devices are mains-powered and **listen continuously** between their own transmissions ([[Continuous-Receive-Downlink]]). ⚠ On the SX1276, continuous RX has the FIFO-pointer trap, which the driver must handle.

## Timing coupling with tier 1

A slave's upstream radios can't transmit while its downstream radios are receiving ([[Dual-Radio-Redundancy-And-Coexistence]]). So the cell superframe reserves a **quiet window** that lines up with the master's tier-1 slot for that slave. Rough plan for a 60 s cycle:

```
0 s     cell beacon (both channels)
0.3 s   contention window (alarms, joins), ~1 s
1.5 s   device uplink slots, ~8–17 s depending on SF and channel count
~19 s   group ACK
~20 s   QUIET: tier-1 exchange with the master (both directions), a few seconds
...     idle / retries / unicast commands / duty-cycle recovery
60 s    next beacon
```

**The master's slot grant for each slave is designed together with this layout.** That's one of the two hardest design items in the system; the other is the cell join and slot assignment.

## Join and slot assignment

- A device joins with a `JOIN_REQ` in the contention window. The slave checks it against the whitelist the master pushed to it, then assigns `{home_channel, slot_index, short_address}` ([[Custom-Protocol-Security]]).
- **The slot table is persisted on the slave** and mirrored to the master, so a slave reboot doesn't force every device to re-join.
- After a mains restore, every device in the section boots at once. **Joins are randomised and back off** (randomness from SX1276 wideband RSSI on the Blue Pill devices), or the contention window collapses. The real fix: a device that already holds a valid slot assignment in its own non-volatile memory **resumes in its slot without re-joining**.

## Related

- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[LoRa-Airtime-Budget]]
- [[Media-Access-Control-Options]]
- [[Custom-Protocol-Frame-Design]]

## Sources

- Client information via the project owner, 2026-10-14: concern about "~50 devices connected at once" and a bottleneck, which motivated the second downstream radio.
- Airtime computed 2026-10-14 with the Semtech time-on-air formula ([[LoRa-Airtime-Budget]]): 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on.

## History

- 2026-10-14: Note created when the client indicated ~50 devices per slave; replaces the earlier "6 endpoint devices" assumption.
- 2026-10-14: One device per pole confirmed; exact count and number of sections not fixed, so a 60-device-per-slave ceiling is used.

Back to [[Index]]
