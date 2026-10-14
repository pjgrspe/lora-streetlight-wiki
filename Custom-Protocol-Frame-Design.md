# Custom Protocol Frame Design

Concrete starting design for the two-tier star. Steal LoRaWAN's structure without adopting LoRaWAN — see [[LoRa-vs-LoRaWAN]].

## Addressing

Three-level hierarchy mirroring the topology:

```
[ NetID : 8 ] [ GwID : 4 ] [ NodeID : 8 ]     — 20-bit logical address
```

- **`NetID`** — site/installation ID. Stops two municipalities' systems cross-talking on the same channel, and is a cheap first-pass filter.
- **`GwID`** — 0 = master, 1..3 = slave gateways, 15 = broadcast to all gateways.
- **`NodeID`** — 0 = the gateway itself, 1..254 = devices under it, 255 = broadcast within that cell.

Carry the 64-bit DevEUI (or the STM32 96-bit UID) **only during provisioning**, never in steady-state frames — at 1758 b/s every byte costs real airtime. See [[Device-Identity-Namespaces]].

Also set a distinct **hardware sync word** per network. It won't prevent a collision but stops your receiver wasting time on foreign traffic.

## Frame format

```
+------+------+------+------+--------+-----+---------+---------+------+
| VER  | TYPE | DST  | SRC  | SEQ    | LEN | HDRCRC  | PAYLOAD | MIC  |
| 4b   | 4b   | 20b  | 20b  | 16b    | 8b  |  8b     |  0..N   | 4 B  |
+------+------+------+------+--------+-----+---------+---------+------+
        \_________________ 12-byte header ________________/
```

- **`VER` first, always.** Four bits of version in byte 0 is the cheapest forward-compatibility insurance available. Field-deployed street lights outlive protocol revisions.
- **`TYPE`** (16 values is plenty): `BEACON, POLL, DATA, ACK, NACK, CMD, CMD_ACK, JOIN_REQ, JOIN_ACC, FRAG, FRAG_ACK, FW_CHUNK, TIME_SYNC, ALARM, RELAY, RSVD`.
- **`SEQ`** — 16-bit, monotonic per source, persisted in NVM. Doubles as the AES-CTR nonce input and the duplicate-suppression key.
- **`HDRCRC`** — CRC-8 over the header only, so a receiver rejects a corrupt header before trusting its length field. The LoRa explicit header CRCs itself, but not *your* application header.
- **`MIC`** — truncated AES-CMAC. Replaces a payload CRC-32 and gives authenticity for free.
- **Little-endian throughout, and say so in the spec.** Mixed endianness is the number-one interop bug in home-grown protocols.

## Payload budget — why SF9 is the design centre

A 12-byte header plus a 4-byte MIC is 16 bytes of overhead. Under a 400 ms dwell limit ([[LoRa-Airtime-Budget]]):

| SF | Max PHY payload | Usable app payload |
|---|---|---|
| 7 | 255 B | 239 B |
| 8 | 138 B | 122 B |
| **9** | **66 B** | **50 B** ← comfortable |
| 10 | 24 B | **8 B** ← tight |
| 11/12 | 0 B | unusable |

If you must shrink for SF10: define a compact DATA type with 12-bit addresses and an 8-bit SEQ, saving 3 bytes.

## CRC / FEC layering

| Layer | Mechanism |
|---|---|
| PHY | LoRa FEC at CR 4/5 + the 16-bit hardware payload CRC — both free, leave on |
| Header | CRC-8 (poly 0x07 or 0x2F) |
| Frame | **AES-CMAC-128 truncated to 4 bytes** — 2⁻³² forgery probability per attempt, same as LoRaWAN |

**Do not** add application-layer Reed-Solomon on top of LoRa's convolutional coding for normal traffic. **Do** consider forward erasure coding for firmware images.

## ACK and retry

- **ACK every command that changes physical state.** Fire-and-forget is unacceptable for street lighting.
- Retry up to 3 times. Timeout must be ≥ `ToA(response) + turnaround + 20%`. At SF9 with a 40 B response: 288 ms + ~30 ms → **use 400 ms**, not a blanket 1 s.
- **Exponential backoff with jitter**: `t_n = t_base · 2^n + U(0, t_base)`. Without jitter, two nodes that collided collide again deterministically — a classic failure.
- **On retry escalate SF, not power** (you are likely already at the legal EIRP ceiling). SF9→SF10→SF11 buys 5 dB at 4x airtime. Cap the escalation so you don't breach the dwell limit.
- **The ACK must echo the SEQ it acknowledges, and be MIC'd.** A bare "OK" is unattributable and replayable.

## Fragmentation

Needed for firmware images, 24-hour schedule tables, and bulk log retrieval. Model it on LoRa Alliance **TS004 Fragmented Data Block Transport** — well designed and free to copy:

- **Session setup first**: `FragIndex`, `FragNb`, `FragSize`, plus a whole-block CRC/hash.
- Fixed fragment size, sized to the SF's dwell-limited payload — 48 B at SF9.
- **Bitmap-based NACK, not per-fragment ACK.** The receiver reports missing fragments as a bitmap after the block; the sender retransmits only gaps. Per-fragment ACK doubles airtime.
- Consider forward erasure coding for multicast firmware, so receivers reconstruct with no uplink at all.

## Related

- [[Custom-Protocol-Security]]
- [[Media-Access-Control-Options]]
- [[Asynchronous-Command-Job-Pattern]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. TS004: https://lora-alliance.org/wp-content/uploads/2020/11/fragmented_data_block_transport_v1.0.0.pdf · TS011 Relay (useful reference for hop addressing): https://resources.lora-alliance.org/technical-specifications/ts011-1-0-0-relay
