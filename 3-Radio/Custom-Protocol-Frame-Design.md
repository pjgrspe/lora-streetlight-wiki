# Custom Protocol Frame Design

The frame format, addressing, ACK/retry and fragmentation rules for the project's own LoRa protocol, used on both tiers. Bottom line: a 12-byte header plus a 4-byte MIC, a version nibble first, little-endian, sized so a useful payload fits SF9 under the 400 ms dwell limit.

## Key points

- **Hierarchical 20-bit address:** `NetID:8 | GwID:4 | NodeID:8`. Any number of slaves up to 14 per site, up to 254 devices per slave.
- **`VER` in the first nibble**, and **little-endian throughout**, written into the spec.
- 16 bytes of overhead leaves **50 B of app payload at SF9** and only 8 B at SF10.
- **ACK every state-changing command**; ACKs echo the `SEQ` and carry a MIC. Retries use jittered exponential backoff and **escalate SF only up to SF10**.
- One frame format and one library for all three roles (master, slave, device). The device role ships to the device developer ([[Team-And-Ownership]]).

## Addressing

Three levels, mirroring the topology:

```
[ NetID : 8 ] [ GwID : 4 ] [ NodeID : 8 ]     — 20-bit logical address
```

- **`NetID`**: site/installation ID. Stops two installations cross-talking on the same channel, and is a cheap first-pass filter.
- **`GwID`**: 0 = master, 1..14 = slave gateways (one per section; the number of sections isn't fixed), 15 = broadcast to all gateways.
- **`NodeID`**: 0 = the gateway itself, 1..254 = devices in that slave's cell (design ceiling 60), 255 = broadcast within the cell.

Carry the long unique ID (DevEUI or the STM32 96-bit UID) **only during provisioning**, never in steady-state frames: at 1758 b/s (SF9) every byte costs airtime. See [[Device-Identity-Namespaces]].

Also set a distinct **hardware sync word** per network. It won't prevent collisions, but it stops receivers wasting time on foreign LoRa traffic.

## Frame format

```
+------+------+------+------+--------+-----+---------+---------+------+
| VER  | TYPE | DST  | SRC  | SEQ    | LEN | HDRCRC  | PAYLOAD | MIC  |
| 4b   | 4b   | 20b  | 20b  | 16b    | 8b  |  8b     |  0..N   | 4 B  |
+------+------+------+------+--------+-----+---------+---------+------+
        \_________________ 12-byte header ________________/
```

| Field | Notes |
|---|---|
| `VER` | **Always first.** Four bits of version is the cheapest forward-compatibility insurance; field-deployed street lights outlive protocol revisions |
| `TYPE` | `BEACON, POLL, DATA, ACK, NACK, CMD, CMD_ACK, JOIN_REQ, JOIN_ACC, FRAG, FRAG_ACK, FW_CHUNK, TIME_SYNC, ALARM, RELAY, RSVD`. `BEACON` also carries the cell slot-map version and pending-command bitmap ([[Slave-Cell-Capacity-And-Scheduling]]) |
| `SEQ` | 16-bit, monotonic per source, persisted in NVM. The low half of a 32-bit counter that feeds the MIC and nonce ([[Custom-Protocol-Security]]); also the duplicate-suppression key, including across a box's two radios |
| `HDRCRC` | CRC-8 over the header only, so a receiver rejects a corrupt header before trusting its length field. The LoRa explicit header protects itself, not *our* header |
| `MIC` | Truncated AES-CMAC. Replaces a payload CRC-32 and adds authenticity |

**Little-endian throughout, and say so in the spec.** Mixed endianness is the number-one interop bug in home-grown protocols, and with a separate device developer it's a real risk.

## Payload budget

16 bytes of overhead (12 B header + 4 B MIC) under the 400 ms dwell limit ([[LoRa-Airtime-Budget]]):

| SF | Max PHY payload | Usable app payload |
|---|---|---|
| 7 | 255 B | 239 B |
| 8 | 138 B | 122 B |
| **9** | **66 B** | **50 B** ← comfortable |
| 10 | 24 B | **8 B** ← tight |
| 11/12 | 0 B | not permitted |

To fit more at SF10, define a compact DATA type with 12-bit addresses and an 8-bit SEQ (saves 3 bytes). A device's scheduled 32 B report (16 B data) fits comfortably at SF9.

## CRC / FEC layering

| Layer | Mechanism |
|---|---|
| PHY | LoRa FEC at CR 4/5 + the 16-bit hardware payload CRC: both free, leave on |
| Header | CRC-8 (poly 0x07 or 0x2F) |
| Frame | **AES-CMAC-128 truncated to 4 bytes**: 2⁻³² forgery probability per attempt, same as LoRaWAN |

**Don't** add application-layer Reed-Solomon on top of LoRa's coding for normal traffic. **Do** consider forward erasure coding for any future firmware images.

## ACK and retry

- **ACK every command that changes physical state.** Fire-and-forget is unacceptable for street lighting. Inside a cell, routine device reports are confirmed by **one group-ACK bitmap per cycle**, not per-frame ACKs.
- Retry up to 3 times. Timeout ≥ `ToA(response) + turnaround + 20%`. At SF9 with a 40 B response that's 288 ms + ~30 ms, so **use 400 ms**, not a blanket 1 s.
- **Exponential backoff with jitter:** `t_n = t_base · 2^n + U(0, t_base)`. Without jitter, two nodes that collided collide again deterministically. Devices get their randomness from the SX1276 wideband RSSI.
- **On retry, escalate SF, not power** (the radios already run at the legal EIRP ceiling). SF9 → SF10 buys ~2.5 dB at 2× airtime; **SF10 is the cap**, since SF11/12 break the dwell limit.
- **Emergency (P0) commands also go out on the backup radio** at tier 1. Receivers drop the duplicate by `SEQ` ([[Dual-Radio-Redundancy-And-Coexistence]]).
- **The ACK must echo the SEQ it acknowledges, and carry a MIC.** A bare "OK" is unattributable and replayable.

## Fragmentation

Needed for 24-hour schedule tables, bulk log retrieval, configuration pushes and any future firmware images. Model it on LoRa Alliance **TS004 Fragmented Data Block Transport**:

- **Session setup first:** `FragIndex`, `FragNb`, `FragSize`, plus a whole-block CRC or hash.
- Fixed fragment size matched to the SF's dwell-limited payload: 48 B at SF9.
- **Bitmap-based NACK, not per-fragment ACK.** The receiver reports missing fragments as a bitmap after the block, and the sender resends only the gaps. Per-fragment ACK doubles the airtime.

## Related

- [[Custom-Protocol-Security]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Asynchronous-Command-Job-Pattern]]
- [[LoRa-vs-LoRaWAN]]

## Sources

- Research synthesis, 2026-09-09. TS004: https://lora-alliance.org/wp-content/uploads/2020/11/fragmented_data_block_transport_v1.0.0.pdf · TS011 Relay (useful reference for hop addressing): https://resources.lora-alliance.org/technical-specifications/ts011-1-0-0-relay

## History

- 2026-10-14: GwID range covers any number of slaves (not fixed at 3); SF escalation capped at SF10 under the adopted 400 ms dwell limit; group-ACK in cells; P0 duplicated on the backup radio; one library shared with the device developer.

Back to [[Index]]
