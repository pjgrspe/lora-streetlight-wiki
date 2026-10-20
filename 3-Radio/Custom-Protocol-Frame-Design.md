# Custom Protocol Frame Design

The frame format, addressing, acknowledgement and fragmentation rules of the project's own LoRa protocol, used on both tiers. Bottom line: an 8-byte header plus a 4-byte MIC (12 bytes of overhead), version first, little-endian, so a useful payload fits SF9 under the 400 ms dwell limit.

## Key points

- **12-bit address `GwID:4 | NodeID:8`** inside an 8-bit site **NetID**. Up to 14 slaves per site; NodeIDs 1–63 per section in v1.
- **8-byte header + 4-byte MIC.** No length field and no header CRC: the LoRa explicit header carries the length and the PHY CRC protects it.
- **54 B of payload at SF9**, 243 B at SF7.
- **Little-endian throughout** on the radio. `VER` sits in the first byte.
- **Routine reports are not acknowledged** (they are snapshots). Commands, events, detail and joins are.
- One frame format and one library for all three roles. The device role ships to the device developer ([[Team-And-Ownership]]).

Specified in `Specs/specs/00-common-frame-and-conventions.md` (v0.1). This note keeps the reasoning.

## Addressing

```
NetID (8 bits, per site)   +   address = GwID:4 | NodeID:8   (12 bits)
```

- **`NetID`**: site ID, 1–254. Frames from another site are dropped before any crypto work. It is sent once per frame, not once per address, because source and destination are always on the same site.
- **`GwID`**: 0 = master, 1–14 = slaves (one per section; the number of sections is configured, not fixed), 15 = all gateways.
- **`NodeID`**: 0 = the gateway itself, 1–63 = devices (60 used at most), 64–253 reserved for growth, 254 = an unjoined device (only in `JOIN_REQ`), 255 = every device in the cell.

The same 12-bit value is the device reference in the Modbus register map, so a pole's address means the same thing on the radio and at the server ([[Modbus-Register-Map-Design]]).

The long unique ID (the STM32 96-bit UID) travels **only in joins**, never in routine frames: every byte costs airtime ([[Device-Identity-Namespaces]]).

## Frame format

```
byte 0     VER:4 | DIR:1 | ACKREQ:1 | PRIO:2
byte 1     TYPE
byte 2     NETID
bytes 3–5  DST:12 | SRC:12
bytes 6–7  FCNT (low 16 bits of a 32-bit counter)
           PAYLOAD (encrypted)
           MIC (4 bytes)
```

| Field | Why it's there |
|---|---|
| `VER` | Always first. Four bits of version is the cheapest forward-compatibility insurance; street lights outlive protocol revisions |
| `DIR` | Uplink/downlink. Also a MIC input, so an uplink can't be replayed as a downlink |
| `ACKREQ`, `PRIO` | Lets every hop handle acknowledgement and priority without parsing the payload |
| `TYPE` | One number space for both tiers (beacon, report, event, command, ack, join, fragments, summary, detail, heartbeat, whitelist). Registry in spec 00 §7 |
| `FCNT` | Low half of the sender's 32-bit counter. Feeds the MIC and the encryption counter, and is the duplicate key, including across a box's two radios ([[Custom-Protocol-Security]]) |
| `MIC` | Truncated AES-CMAC. Replaces a payload CRC and adds authenticity |

**Why no length or header CRC:** the earlier design had both (a 12-byte header). The LoRa explicit header already carries the PHY length, and the PHY CRC drops corrupt frames in hardware. The MIC catches anything that slips through. Dropping both saves 4 bytes of airtime on every frame.

**Little-endian, and the spec says so.** Mixed endianness is the number-one interop bug in home-grown protocols, and with a separate device developer it's a real risk. (Modbus stays big-endian, high word first: that's a different interface.)

## Payload budget

12 bytes of overhead under the 400 ms dwell limit ([[LoRa-Airtime-Budget]]):

| SF | Max PHY payload | Max frame payload |
|---|---|---|
| 7 | 255 B | 243 B |
| 8 | 138 B | 126 B |
| **9** | **66 B** | **54 B** |
| 10 | 24 B | 12 B (not used in v1) |

An LED device's 20-byte report is a 32-byte frame (247 ms at SF9). The meter's 44-byte report is 56 bytes (349 ms) and takes a double slot.

## CRC / FEC layering

| Layer | Mechanism |
|---|---|
| PHY | LoRa FEC at CR 4/5 + the 16-bit hardware payload CRC: both on |
| Frame | **AES-CMAC-128 truncated to 4 bytes**: 2⁻³² forgery probability per attempt, as in LoRaWAN |

Don't add application-layer Reed-Solomon for normal traffic. Consider forward erasure coding only for any future firmware images.

## Acknowledgement and retry

- **Commands that change physical state are always acknowledged** with a `CMD_ACK` that carries the resulting state and a result code from one system-wide table (spec 02 §11.5).
- **Routine device reports are not acknowledged.** Each is a snapshot; a lost one is replaced by the next cycle's. **Events** are acknowledged through the cell beacon's event-ACK bitmap. Tier-1 summaries are acknowledged in the master's beacon.
- **Retries use jittered exponential backoff** so two nodes that collided don't collide again. Devices get randomness from the SX1276 wideband RSSI.
- **No spreading-factor escalation inside a scheduled cell**: slot lengths are fixed. A link that doesn't close at SF9 is a placement problem ([[Link-Budget-And-Site-Survey]]).
- **P0 commands go out on both tier-1 radios and are repeated once 1 s later.** The copies are byte-identical, with the same frame counter, so receivers drop the duplicates ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Every hop also drops a repeated `cmd_id`, so a retried command is acknowledged as a duplicate but never executed twice.

## Fragmentation

For anything larger than one frame: whitelist dumps, configuration blobs, logs. Modelled on LoRa Alliance TS004:

- Fragment 0 carries a block header (content type, length, CRC-32 of the whole block).
- Fixed fragment size per SF: 232 B at SF7, 120 B at SF8, 48 B at SF9.
- **Bitmap NACK, not per-fragment ACK.** The receiver reports missing fragments after the last one; the sender resends only the gaps, up to 3 rounds.

Ordinary commands, including schedule changes, fit in one frame and don't fragment.

## Related

- [[Custom-Protocol-Security]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Asynchronous-Command-Job-Pattern]]
- [[LoRa-vs-LoRaWAN]]

## Sources

- `Specs/specs/00-common-frame-and-conventions.md`, `02-tier1-protocol.md`, `03-tier2-device-protocol.md` (v0.1, 2026-10-21)
- Research synthesis, 2026-09-09. TS004: https://lora-alliance.org/wp-content/uploads/2020/11/fragmented_data_block_transport_v1.0.0.pdf · TS011 Relay: https://resources.lora-alliance.org/technical-specifications/ts011-1-0-0-relay

## History

- 2026-10-14: GwID range covers any number of slaves (not fixed at 3); SF escalation capped at SF10; group-ACK in cells; P0 duplicated on the backup radio; one library shared with the device developer.
- 2026-10-21: aligned with spec pack v0.1. Header cut from 12 to 8 bytes (12-bit addresses inside one NetID, no length field, no header CRC); 54 B payload at SF9; reports unacknowledged, events acked by bitmap; no SF escalation in cells; P0 copies share one frame counter.

Back to [[Index]]
