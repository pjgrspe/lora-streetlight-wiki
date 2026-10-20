# Custom Protocol Security

How the LoRa protocol is protected against forged and replayed commands. Bottom line: **copy LoRaWAN's crypto design; don't invent your own.** A recorded "all lights off" frame that replays forever is a public-safety problem, and pole-mounted hardware is physically reachable by attackers.

## Key points

- **AES-128-CMAC (4-byte MIC)** on every frame, **AES-128-CTR** for payload confidentiality, a **32-bit frame counter** in the MIC, and a direction bit.
- **Verify the MIC before decrypting or parsing.** Failures are dropped silently and only counted.
- **Keys derive from one `SiteKey` held only by the gateways.** Each device's key is `DevKey = CMAC(SiteKey, UID)`, so a slave needs no per-device key table, and a stolen device reveals only its own key.
- **Fresh session keys at every join and rejoin.** After a reboot a device rejoins **in its own slot**, so re-keying doesn't cause a join storm.
- **Beacons and the tier-1 heartbeat are MIC'd but not encrypted**, so an unjoined node can read the timing.
- **Crypto runs in software** on every node (no AES hardware on the L476 or the F103), once, in the shared library.
- Message authentication and replay protection are in the wire format from v1. Full key management is a later hardening step.

Specified in `Specs/specs/00-common-frame-and-conventions.md` §6 (v0.1).

## The scheme

| Key | Held by | How it's made | Protects |
|---|---|---|---|
| `SiteKey` | Master and every slave, loaded over the service console. Never on a device, never over the air | Random, at commissioning | Root of everything |
| `DevKey` | One device + its slave (+ master) | `AES-CMAC(SiteKey, 0x44 ‖ UID)` | Device joins |
| `GwKey(n)` | Slave n + master | `AES-CMAC(SiteKey, 0x47 ‖ n)` | Slave joins |
| `NwkSKey`, `AppSKey` | Session pair per device and per slave | Derived at join from `DevKey`/`GwKey`, the join nonces and the NetID | MIC and encryption of unicast frames |
| `CellKey(n)` | Slave n and its joined devices | Random per slave, persisted; delivered inside the encrypted join accept | Cell broadcasts (beacon, group commands) |
| `NetKey` | Master and all joined slaves | Random, persisted; delivered in the slave's join accept | Tier-1 broadcasts |

| Mechanism | Rule |
|---|---|
| **Integrity** | 4-byte truncated AES-CMAC over a 16-byte B0 block (direction, NetID, addresses, 32-bit counter, length), the header and the encrypted payload. **Verified before decrypting**, in constant time |
| **Confidentiality** | AES-128-CTR. The counter block includes direction, NetID, both addresses and the 32-bit counter, so it's **unique per (key, block)** within a session |
| **Replay** | 32-bit counter; only the low 16 bits are sent, but **all 32 feed the MIC**. The receiver keeps the last accepted value plus a 32-entry bitmap; a repeat is acknowledged but never executed twice |
| **Direction** | A `DIR` bit in the header and in the MIC input. **Without it, an attacker can reflect an uplink back as a downlink** |
| **P0 copies** | The copies of a P0 frame on both radios, and its 1 s repeat, are byte-identical with the **same** counter, so the duplicate window drops them. Every other retransmission takes a new counter |

**Why derived keys:** a slave needs only `SiteKey` to validate any whitelisted device, and the master pushes just UIDs to it. ⚠ The cost: a stolen *gateway* reveals `SiteKey`. Gateways sit in locked cabinets with readout protection level 1. Stored per-device keys are the hardening path.

## Platform realities

| Node | MCU | Crypto | Randomness |
|---|---|---|---|
| Master, slaves | NUCLEO-L476RG class | **Software** AES-CMAC/CTR | Hardware RNG on the L476 |
| Devices | STM32F103 (Blue Pill) | **Software** AES, inside the device library (≤16 KB flash total) | **SX1276 wideband RSSI** low bits, seeded with the chip UID |

- Software AES on a 72 MHz M3 or 80 MHz M4 is easily fast enough for a few frames per cycle.
- **`DevNonce` is a persisted counter**, so the join itself needs no randomness. Randomness is only for backoff jitter.
- **Counters survive reboots:** a gateway writes a high-water mark to FRAM every 64 frames, a device to flash every 256 frames (F103 flash wears out after ~10k erases). After a reboot the sender jumps forward by that step. A sender near counter exhaustion must rejoin.

## The ten things custom protocols get wrong

1. **Nonce/counter reuse: the catastrophic one.** In CTR mode, reusing a counter block with the same key gives `C1 ⊕ C2 = P1 ⊕ P2`, recovering plaintext without the key. A documented LoRaWAN failure. Mitigations here: fresh session keys at every join/rejoin, plus high-water marks written ahead of use.
2. **CRC mistaken for integrity.** A CRC is linear; an attacker flips bits and fixes it. Only a keyed MAC gives authenticity.
3. **MAC-then-decrypt inverted.** Verify the MIC *before* decrypting and parsing. Decrypting attacker-controlled bytes into a parser is how remote code execution happens.
4. **No replay protection on commands.** Without the counter in the MIC, a recorded frame replays forever.
5. **One shared key for the whole fleet.** Extract one node and you own every light. Per-device keys, here by derivation; the site key never touches a device.
6. **Keys in unprotected flash, the git repo or factory test firmware.** Readout protection level 1 on gateways and devices. Zeroise keys in RAM after use; never log one. ⚠ **Never RDP Level 2**: it's irreversible and kills field debugging. ⚠ Clone F103 chips may not implement readout protection faithfully.
7. **Unauthenticated join.** `JOIN_REQ` is MIC'd with `DevKey` and carries a strictly increasing persisted `DevNonce`; the slave rejects a non-increasing one. `JOIN_ACC` is encrypted and MIC'd and carries a fresh join nonce, so a replayed join can't reset session keys.
8. **No firmware-image authentication.** Unauthenticated updates are a remote-bricking primitive. Sign images and verify in an immutable bootloader ([[Firmware-Update-Over-LoRa]]). (Not in the current engagement.)
9. **Timing and error oracles.** No distinguishable errors for bad MIC vs bad counter vs unknown device; MIC failures get no reply at all. Constant-time comparison.
10. **No key-rotation path.** Plan it from day one even if unused. A deployment that can't rotate keys has a single-event lifetime.

## Pairing and provisioning

Threat model: physical access to poles.

1. **Bench:** each device has its STM32 UID and a QR label. Its `DevKey` follows from the UID and the site key, so nothing secret is stored in a provisioning database for devices.
2. **Commissioning:** a technician scans the QR code into the server, which authorises that UID for a section. The master pushes the section's whitelist to the slave. **No open join**: a new UID is only accepted while the slave's commissioning window is open.
3. **Join:** the device sends `JOIN_REQ{UID, DevNonce, type, firmware}` in the contention window. The slave derives `DevKey`, checks the MIC and the whitelist, and replies with an encrypted `JOIN_ACC{join nonce, NodeID, channel, slot, cell parameters, CellKey}`. Both sides derive session keys.
4. **Rejoin** after a reboot, on request, or on counter exhaustion: a device that holds a slot sends `REJOIN_REQ` in **its own slot**, so a section-wide mains restore re-keys every device without contention.
5. **Decommissioning:** revoke the UID so a stolen unit can't rejoin. A swapped device's old UID goes on the refused list.

## Security scope at this stage

Full key management (provisioning tooling, rotation, production readout-protection procedure) is a later hardening add-on. But **the frame counter, the MIC and the direction bit are in the wire format from v1.** Retrofitting anti-replay after a hundred-plus devices are installed is painful, and therefore gets skipped.

## Related

- [[Custom-Protocol-Frame-Design]]
- [[LoRa-vs-LoRaWAN]]
- [[Commissioning-And-Field-Swap]]
- [[STM32-Part-Selection]]

## Sources

- `Specs/specs/00-common-frame-and-conventions.md` §6, `03-tier2-device-protocol.md` §4–5 (v0.1, 2026-10-21)
- Research synthesis, 2026-09-09. LoRaWAN security white paper: https://lora-alliance.org/wp-content/uploads/2020/11/lorawan_security_whitepaper.pdf · counter-reset vulnerability: https://arxiv.org/pdf/2006.16554 · https://www.tarlogic.com/blog/lorawan-vulnerabilities-versions/ · smart-lighting attack analysis: https://arxiv.org/pdf/2510.20494

## History

- 2026-09-09: Written assuming the STM32WL's hardware AES, PKA and TRNG.
- 2026-10-14: Platforms changed to STM32 Nucleo-class gateways + Blue Pill devices with SX1276, so crypto is software and device randomness comes from radio RSSI. Crypto lives in the shared library delivered to the device developer.
- 2026-10-21: aligned with spec pack v0.1. Key hierarchy fixed (SiteKey on gateways only, derived DevKey/GwKey, session keys at join, CellKey/NetKey for broadcasts); beacons and heartbeat MIC-only; P0 copies share one counter; high-water marks every 64 (gateway) / 256 (device) frames; rejoin in the device's own slot.

Back to [[Index]]
