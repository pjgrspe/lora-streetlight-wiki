# Slave Cell Capacity And Scheduling

How one Slave Gateway serves all the LoRa devices in its section (one per pole, up to 60) without becoming a bottleneck. Bottom line: **the MAC design matters more than the radio count.** Polling can't work under a 1% duty cycle. Scheduled uplink does, and the second downstream radio halves the uplink period.

## Key points

- **Design ceiling: 60 devices per slave** (LED devices plus the section's meter). One device per pole is confirmed; the exact count per section isn't known.
- **Polling is rejected:** a 53-device sweep at SF9 needs **≥15 minutes** under 1% duty, because the slave must transmit once per device.
- **Scheduled uplink (cell TDMA):** one beacon per channel per cycle, then every device reports in its own 0.3 s slot. 64 slot positions, 32 per channel at the same times. Uplink takes **9.6 s on two channels, 19.2 s on one**, inside a **120 s cycle**.
- **Reports are not acknowledged** (they're snapshots). Only events are, through a bitmap in the next beacon.
- **Devices listen all the time**, so commands go out any time in the downlink window; group commands are broadcast with `execute_at`.
- A **quiet window** at 12–16 s holds the slave's tier-1 slot. After a power cut, devices rejoin **in their own slot**, so there's no join storm.

Specified in `Specs/specs/03-tier2-device-protocol.md` and `04-channel-plan-and-airtime.md` (v0.1).

## Cell sizes

| Section (as drawn) | Lights | Devices on the slave's radios |
|---|---|---|
| 1 | 53 | ~54 (53 LED devices + 1 meter) |
| 2 | 38 | ~39 |
| 3 | 37 | ~38 |
| **Any section (design ceiling)** | — | **60** |

The diagram shows three sections, but there may be four or more. The firmware and analysis are per cell and don't depend on the number of sections.

## Option 1: the slave polls each device (rejected)

16 B poll, 32 B reply, ~60 ms turnaround, 53 devices:

| SF | Per device | Sweep | Slave TX per sweep | **Fastest legal sweep at 1% duty** |
|---|---|---|---|---|
| 7 | 183 ms | 9.7 s | 2.7 s | **~4.5 min** |
| 9 | 472 ms | 25 s | 8.7 s | **~15 min** |

The slave transmits once for every device, so **its own duty cycle sets the floor**. Extra radios don't help, because the 1% budget is accounted per box ([[Dual-Radio-Redundancy-And-Coexistence]]). This is the bottleneck the client was worried about, and it comes from the protocol, not the radio count.

## Option 2: scheduled uplink (cell TDMA), chosen

The slave sends **one beacon per channel per cycle** (time, cycle number, slot-map and config versions, event-ACK bitmap). Each device has a fixed slot assigned at join and transmits its report there, with no poll and no acknowledgement.

```
cell time (s)
0.00    BEACON on channel A
0.25    BEACON on channel B
0.60    contention window, 1 s: events, joins
1.60    uplink slots: 32 per channel, 0.3 s each      → ends 11.2
12–16   QUIET: the slave's tier-1 slot with the master
16+     downlink window: commands, join accepts, fragments, retries
120     next cycle
```

- **Slots:** positions 0–31 run on channel A and 32–63 on channel B, at the same times. A 32-byte LED report at SF9 is 247 ms, so a 0.3 s slot leaves a ~50 ms guard. The meter's 56-byte report (349 ms) takes **two consecutive slots** on one channel. 60 devices with one meter need 61 positions.
- **Why reports aren't acknowledged:** every report is the device's full current state. A lost one is replaced by the next, and the device shows as stale after a missed cycle and offline after 3. Acknowledging routine reports would cost the slave airtime it doesn't have ([[LoRa-Airtime-Budget]]).
- **Events** (lamp fault, restart, phase loss) go in the contention window with slotted-ALOHA backoff. The next beacon's event-ACK bitmap confirms them.
- **Why 120 s and not 60 s:** the cell is easily fast enough for 60 s, but the slave's beacons on two channels plus its upstream detail would need ~112% of a per-box 1% budget. At 120 s it's 66% ([[LoRa-Airtime-Budget]]).

**Cadence:** every device reports once per cycle (120 s). The meter reports every cycle; the slave forwards meter data upstream every 5 min. Per-device detail goes to the master every 15 min (basic) and hourly (extended), and immediately for any device whose status changed.

## Single-channel mode

If one downstream radio fails, the surviving channel runs **all 64 positions back to back**: 19.2 s, ending at 20.8 s. The quiet window moves to 21–26 s, and **the slave starts its cell cycle 10 s earlier** so its tier-1 slot, which never moves, lands inside it. Devices that hear the beacon on the other channel with `single_channel_mode` set move over and keep their slot number; no rejoin is needed.

## Commands into the cell

- **Devices listen continuously** between their own transmissions ([[Continuous-Receive-Downlink]]), so a command needs no announcement. The slave sends it in the downlink window (from 16 s), or immediately for P0.
- **Group and section commands are broadcast with `execute_at`** on both channels and repeated once 1 s later, so every device switches at the same instant ([[Popcorn-Effect-And-Scheduled-Execution]]).
- **Confirmation for a broadcast comes from the next reports**, which carry `level_actual` and the low byte of the last executed command's counter. No per-device command-ACK storm.
- **Unicast commands** (dim one pole) are acknowledged straight away with a `CMD_ACK` carrying the resulting state.
- **P0 pre-empts:** the slave broadcasts immediately, even during the uplink period. Reports lost to that aren't retried; the next cycle replaces them.

## Timing coupling with tier 1

A slave's upstream radios can't transmit while its downstream radios are receiving ([[Dual-Radio-Redundancy-And-Coexistence]]). So each slave starts its cell cycle at a fixed offset from master time:

```
cell_start(n) = ( 2 + 2(n − 1) − 13 ) mod CYCLE
```

That puts slave n's 2-second tier-1 slot at cell time 13–15 s, inside the 12–16 s quiet window, and keeps every slave's beacons clear of the master's beacon. The tier-1 side is in [[Media-Access-Control-Options]].

## Join and slot assignment

- A new device listens for beacons from its site, then sends a `JOIN_REQ` (UID, nonce, device type) in the contention window. The slave checks the UID against the whitelist the master pushed to it, then answers with `{NodeID, home channel, slot, cell parameters, CellKey}` ([[Custom-Protocol-Security]]).
- **The same UID always gets the same NodeID back.** The slot table is persisted on the slave in FRAM and mirrored to the master, so a slave reboot doesn't force every device to rejoin.
- **After a mains restore, every device in the section boots at once.** A device that holds a persisted slot sends a `REJOIN_REQ` **in its own slot**, not the contention window. No collisions, by construction. Only devices without a slot use the contention window, with randomised backoff (randomness from the SX1276 wideband RSSI).
- A new UID is only accepted while the slave's commissioning window is open.

## Related

- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[LoRa-Airtime-Budget]]
- [[Media-Access-Control-Options]]
- [[Custom-Protocol-Frame-Design]]

## Sources

- `Specs/specs/03-tier2-device-protocol.md`, `04-channel-plan-and-airtime.md`, `02-tier1-protocol.md` (v0.1, 2026-10-21)
- Client information via the project owner, 2026-10-14: concern about "~50 devices connected at once" and a bottleneck, which motivated the second downstream radio.
- Airtime computed with the Semtech time-on-air formula ([[LoRa-Airtime-Budget]]): 125 kHz, CR 4/5, 8-symbol preamble, explicit header, CRC on.

## History

- 2026-10-14: Note created when the client indicated ~50 devices per slave; replaces the earlier "6 endpoint devices" assumption.
- 2026-10-14: One device per pole confirmed; exact count and number of sections not fixed, so a 60-device-per-slave ceiling is used.
- 2026-10-21: aligned with spec pack v0.1. Cycle 120 s; 64 slot positions of 0.3 s (meter double slot); reports unacknowledged, events acked by beacon bitmap; no pending-command bitmap (devices listen continuously); quiet window 12–16 s with the cell_start offset formula; single-channel mode shifts the cell 10 s earlier; rejoin in the device's own slot.

Back to [[Index]]
