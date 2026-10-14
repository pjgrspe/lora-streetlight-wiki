# Custom Protocol Security

> **Updated 2026-10-14.** The note below assumes STM32WL hardware AES/PKA/TRNG. On the likely STM32G0B1 gateway MCU there is **no AES or TRNG hardware** ([[STM32-Part-Selection]]), so AES-CMAC/CTR runs in software. That's fine at these frame rates, but entropy for nonces needs another source (or use G0C1). Devices are written by another developer: the **device-role MAC library** must carry the crypto, so it's implemented once.

**Copy LoRaWAN's crypto design; do not invent your own.** A recorded "all lights off" frame that replays forever is a public-safety issue, not a data-privacy one. Street-light hardware sits on a publicly reachable pole and is physically attackable.

## The scheme

| Layer | Mechanism |
|---|---|
| **Root secret** | 128-bit `MasterKey`, **per device**, provisioned at bench/factory. Never transmitted. |
| **Session keys** | `NwkSKey = AES-128(MasterKey, 0x01 ‖ DevEUI ‖ JoinNonce ‖ pad)`, `AppSKey` with `0x02`. Derived at join, **discarded on reboot, re-joined**. |
| **Confidentiality** | AES-128-CTR over the payload. Counter block = `Dir ‖ DevAddr ‖ FCnt(32) ‖ BlockIndex` — **must be unique per (key, block) forever.** |
| **Integrity** | AES-128-CMAC over header + 32-bit counter + ciphertext, truncated to 4 bytes. **Verify before decrypting.** |
| **Replay** | 32-bit counter, of which only the low 16 bits are transmitted but **all 32 feed the MIC** — so an attacker cannot roll the counter without breaking the MAC. Plus a 32-entry sliding-window bitmap. |
| **Direction** | A `Dir` bit (0 = uplink, 1 = downlink) in the counter block and MIC input. **Without it, an attacker reflects an uplink back as a downlink.** |

The STM32WL has hardware AES, PKA and TRNG on die — exactly what this needs. See [[STM32-Part-Selection]].

## The ten things custom protocols get wrong

1. **Nonce/counter reuse — the catastrophic one.** In CTR mode, reusing a counter block with the same key gives `C1 ⊕ C2 = P1 ⊕ P2` — plaintext recovery with no key. This is a *documented* LoRaWAN failure: reset the frame counter without changing the key and the cipher recreates the same keystream. Mitigations: (a) persist the counter to NVM with a **high-water mark written ahead of use**, so a power cut can only skip forward; (b) **derive fresh session keys on every reboot/re-join**, which makes (a) non-critical; (c) refuse to transmit at counter max — re-join instead. **Do (b).**
2. **CRC mistaken for integrity.** CRC is linear; an attacker flips payload bits and fixes the CRC trivially. Only a keyed MAC gives authenticity.
3. **MAC-then-decrypt inverted.** Verify the MIC *before* decryption and *before* parsing. Decrypting attacker-controlled bytes into a parser is how you get remote code execution.
4. **No replay protection on commands.** Without the counter in the MIC input, a recorded frame replays forever.
5. **One shared key for the whole fleet.** Extract one pole-mounted node and you own the city. **Per-device root keys, always.** If a per-device key database is impractical, diversify: `DevKey = AES-CMAC(FleetMasterKey, DevEUI)` — but then the fleet master key is the crown jewel and must never live on a node.
6. **Keys in unprotected flash, in the git repo, or in factory test firmware.** Use STM32 readout protection (RDP level 1) and hide-protection on the bootloader region. Zeroise key material in RAM after use. Never log a key. ⚠ **Do not set RDP level 2** — it is irreversible and kills RMA and field debug.
7. **Unauthenticated join.** `JOIN_REQ` must be MIC'd with the root key, and `JOIN_ACCEPT` must carry a server nonce so a replayed join cannot force a session-key reset. LoRaWAN 1.0 got this partly wrong and 1.1 fixed it — **implement the 1.1 behaviour**: a monotonically increasing `DevNonce` persisted in NVM, rejecting any non-increasing nonce.
8. **No firmware-image authentication.** Unauthenticated FUOTA is a remote-bricking primitive across a whole city. **Sign images (ECDSA P-256) and verify in an immutable bootloader.** The transport MIC authenticates the *link*, not the *vendor*. See [[Firmware-Update-Over-LoRa]].
9. **Timing and error oracles.** Don't return distinguishable errors for "bad MIC" vs "bad counter" vs "unknown device". Use a constant-time comparison — `memcmp` is not one.
10. **No key rotation path.** Plan it on day one even if never used. A deployment that cannot rotate keys has a single-event lifetime.

## Pairing and provisioning

Threat model: physical access to poles.

1. **Bench/factory** — burn a unique `DevEUI` + `MasterKey` into protected flash; record the pair in an offline encrypted provisioning database. **Never ship a default key.**
2. **Commissioning** — a technician scans a QR code (DevEUI + short claim code) into the server, which authorises that DevEUI for a specific section. **Whitelist by DevEUI; no open join.**
3. **Join** — device sends `JOIN_REQ{DevEUI, DevNonce, MIC}`; the slave relays to master; master checks the whitelist and nonce monotonicity, replies `JOIN_ACCEPT{JoinNonce, NetID, DevAddr, GwID, NodeID}` MIC'd and encrypted with the root key. Both sides derive session keys.
4. **Re-join** on reboot, on a MIC-failure storm, or on counter exhaustion.
5. **Decommissioning** — revoke the DevEUI so a stolen unit cannot rejoin. Street-light hardware does get stolen.

## Pilot-stage advice

Key management can stay crude at pilot, but **put the frame counter and the MIC in the Phase 0 wire format**. Retrofitting anti-replay after 128 lights are live is painful, and therefore will be skipped.

## Related

- [[Custom-Protocol-Frame-Design]]
- [[Commissioning-And-Field-Swap]]
- [[Development-Roadmap-And-Gates]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. LoRaWAN security white paper: https://lora-alliance.org/wp-content/uploads/2020/11/lorawan_security_whitepaper.pdf · counter-reset vulnerability: https://arxiv.org/pdf/2006.16554 · https://www.tarlogic.com/blog/lorawan-vulnerabilities-versions/ · smart-lighting-specific attack analysis: https://arxiv.org/pdf/2510.20494
