# Custom Protocol Security

How the LoRa protocol is protected against forged and replayed commands. Bottom line: **copy LoRaWAN's crypto design; don't invent your own.** A recorded "all lights off" frame that replays forever is a public-safety problem, and pole-mounted hardware is physically reachable by attackers.

## Key points

- **AES-128-CMAC (4-byte MIC)** on every frame, **AES-128-CTR** for confidentiality, a **32-bit frame counter** in the MIC, and a direction bit.
- **Verify the MIC before decrypting or parsing.**
- **Fresh session keys on every reboot or re-join.** This is what makes counter reuse safe.
- **Crypto runs in software** on every node: no AES hardware on the gateway MCU (STM32L476/G0) or on the devices (STM32F103). Fine at these frame rates.
- **Per-device root keys**, provisioned on the bench. A whitelisted join, no open join.
- The crypto lives **in the shared protocol library**, implemented once for the master, slave and device roles.
- At this stage the MIC and frame counter are in the wire format; full key management is a later hardening step.

## The scheme

| Layer | Mechanism |
|---|---|
| **Root secret** | 128-bit `MasterKey`, **per device**, provisioned on the bench. Never transmitted |
| **Session keys** | `NwkSKey = AES-128(MasterKey, 0x01 ‖ DevEUI ‖ JoinNonce ‖ pad)`, and `AppSKey` with `0x02`. Derived at join, **discarded on reboot**, then re-joined |
| **Confidentiality** | AES-128-CTR over the payload. Counter block = `Dir ‖ DevAddr ‖ FCnt(32) ‖ BlockIndex`, which **must be unique per (key, block) forever** |
| **Integrity** | AES-128-CMAC over header + 32-bit counter + ciphertext, truncated to 4 bytes. **Verify before decrypting** |
| **Replay** | 32-bit counter, of which only the low 16 bits are sent ([[Custom-Protocol-Frame-Design]] `SEQ`), but **all 32 feed the MIC**, so an attacker can't roll the counter without breaking the MAC. Plus a 32-entry sliding-window bitmap |
| **Direction** | A `Dir` bit (0 = uplink, 1 = downlink) in the counter block and MIC input. **Without it, an attacker can reflect an uplink back as a downlink** |

## Platform realities

| Node | MCU | Crypto | Randomness |
|---|---|---|---|
| Master, Slaves | STM32 Nucleo-class (L476 recommended) | **Software** AES-CMAC/CTR | L476 has a hardware RNG; a G0B1 does not |
| Devices | STM32F103 (Blue Pill) | **Software** AES-CMAC/CTR, sized into the device library (≤16 KB flash total) | **SX1276 wideband RSSI** low bits, seeded with the chip UID |

- Software AES on a 72 MHz M3 or 80 MHz M4 is easily fast enough for a few frames per minute.
- **`DevNonce` is a persisted monotonic counter**, so the join needs no randomness. Randomness is only for backoff jitter and session nonces.
- The F103's flash wears out after ~10k erase cycles, so the device **persists the counter high-water mark every N frames** (e.g. 256) and skips forward on reboot. Because session keys are fresh on every reboot, this is defence in depth.

## The ten things custom protocols get wrong

1. **Nonce/counter reuse: the catastrophic one.** In CTR mode, reusing a counter block with the same key gives `C1 ⊕ C2 = P1 ⊕ P2`, recovering plaintext without the key. This is a *documented* LoRaWAN failure. Mitigations: (a) persist the counter with a **high-water mark written ahead of use**; (b) **derive fresh session keys on every reboot or re-join**; (c) refuse to transmit at counter maximum and re-join instead. **Do (b)**; (a) is the backstop.
2. **CRC mistaken for integrity.** A CRC is linear; an attacker flips payload bits and fixes the CRC trivially. Only a keyed MAC gives authenticity.
3. **MAC-then-decrypt inverted.** Verify the MIC *before* decrypting and *before* parsing. Decrypting attacker-controlled bytes into a parser is how remote code execution happens.
4. **No replay protection on commands.** Without the counter in the MIC input, a recorded frame replays forever.
5. **One shared key for the whole fleet.** Extract one pole-mounted node and you own every light. **Per-device root keys, always.** If a per-device key database is impractical, diversify: `DevKey = AES-CMAC(FleetMasterKey, DevEUI)`. But then the fleet master key is the crown jewel and must never be on a device.
6. **Keys in unprotected flash, in the git repo, or in factory test firmware.** Use STM32 readout protection (RDP Level 1) on gateways and devices. Zeroise key material in RAM after use. Never log a key. ⚠ **Never RDP Level 2**: it's irreversible and kills RMA and field debugging. ⚠ Clone F103 chips may not implement readout protection faithfully.
7. **Unauthenticated join.** `JOIN_REQ` must be MIC'd with the root key, and `JOIN_ACCEPT` must carry a server nonce, so a replayed join can't force a session-key reset. **Implement LoRaWAN 1.1 behaviour**: a monotonically increasing `DevNonce` persisted in NVM, rejecting any non-increasing value.
8. **No firmware-image authentication.** Unauthenticated firmware updates are a remote-bricking primitive. **Sign images (ECDSA P-256) and verify in an immutable bootloader** ([[Firmware-Update-Over-LoRa]]). The transport MIC authenticates the *link*, not the *vendor*.
9. **Timing and error oracles.** Don't return distinguishable errors for "bad MIC" vs "bad counter" vs "unknown device". Use a constant-time comparison; `memcmp` isn't one.
10. **No key-rotation path.** Plan it from day one even if it's never used. A deployment that can't rotate keys has a single-event lifetime.

## Pairing and provisioning

Threat model: physical access to poles.

1. **Bench:** burn a unique `DevEUI` + `MasterKey` into protected flash, and record the pair in an offline, encrypted provisioning database. **Never ship a default key.**
2. **Commissioning:** a technician scans a QR code (DevEUI + short claim code) into the server, which authorises that DevEUI for a specific section. **Whitelist by DevEUI; no open join.**
3. **Join:** the device sends `JOIN_REQ{DevEUI, DevNonce, MIC}` in its cell's contention window. The **master owns the whitelist** and pushes each section's entries to its slave, so the slave can validate and assign a slot locally ([[Slave-Cell-Capacity-And-Scheduling]]). `JOIN_ACCEPT{JoinNonce, NetID, DevAddr, GwID, NodeID, slot}` is MIC'd and encrypted with the root key, and both sides derive session keys.
4. **Re-join** on reboot, on a burst of MIC failures, or on counter exhaustion. A device that holds a valid slot assignment resumes in its slot after re-keying, to avoid a mass re-join storm after mains restore.
5. **Decommissioning:** revoke the DevEUI so a stolen unit can't rejoin. Street-light hardware does get stolen.

## Security scope at this stage

Full key management (provisioning tooling, rotation, production readout protection) is a hardening step for later. But **the frame counter, the MIC and the direction bit go into the wire format from the first version.** Retrofitting anti-replay after a hundred-plus devices are installed is painful, and therefore gets skipped.

## Related

- [[Custom-Protocol-Frame-Design]]
- [[LoRa-vs-LoRaWAN]]
- [[Commissioning-And-Field-Swap]]
- [[STM32-Part-Selection]]

## Sources

- Research synthesis, 2026-09-09. LoRaWAN security white paper: https://lora-alliance.org/wp-content/uploads/2020/11/lorawan_security_whitepaper.pdf · counter-reset vulnerability: https://arxiv.org/pdf/2006.16554 · https://www.tarlogic.com/blog/lorawan-vulnerabilities-versions/ · smart-lighting-specific attack analysis: https://arxiv.org/pdf/2510.20494

## History

- 2026-09-09: Written assuming the STM32WL's hardware AES, PKA and TRNG.
- 2026-10-14: Platforms changed to STM32 Nucleo-class gateways + Blue Pill devices with SX1276, so crypto is software and device randomness comes from radio RSSI. Crypto lives in the shared library delivered to the device developer.

Back to [[Index]]
