# Firmware Update Over LoRa

How firmware gets onto the gateways and devices after installation, and how to avoid bricking them doing it. The bottom line: the Master updates over RS485, the Slaves can update over LoRa slowly, and per-pole devices over LoRa are only practical with multicast. Over-the-air update is **not in the current contracted scope**: prototype gateways are updated over SWD/RS485.

## Key points

- **Not in current scope.** Prototype gateways are flashed over SWD (on-board ST-LINK) or RS485. This note is the design for when OTA is added.
- **The NUCLEO-L476RG has dual-bank flash**, so the simple bank-swap path is available: write the new image to the inactive bank, verify, swap, and roll back by swapping again. External NOR staging becomes optional.
- **Sign every image (ECDSA P-256) and verify in an immutable boot stage.** Include `target_hw_id`, because master, slave and device images share one radio network.
- **The Master updates over RS485, never over LoRa.**
- **Under the ≤1% per-box duty cycle, LoRa updates are slow:** hours per Slave, and only multicast with erasure coding is viable for the per-pole devices.

## Update path on the gateways (STM32L4 dual-bank)

The L476's 1 MB flash is two 512 KB banks, and the boot bank is selectable by option bit (⚠ verify the exact mechanism, `BFB2`, against the L4 reference manual). The application fits easily in one bank.

```
Bank 1 (512 KB)  running image          Bank 2 (512 KB)  new image written here
                                          ↓ verify SHA-256 + ECDSA signature
                                          ↓ set "pending" record in FRAM, swap boot bank, reset
                                          ↓ new image must call mark_healthy() in time
                                          ✗ not healthy after 3 attempts → swap back
```

1. The app receives the image into the **inactive bank**, chunk by chunk.
2. It verifies the whole image: **SHA-256 + ECDSA-P256 signature**.
3. It writes an "install pending" record to FRAM (`attempts = 0`) and swaps banks.
4. The new image must call `mark_healthy()` within ~10 minutes, gated on **a successful round trip on its links** (Modbus *and* LoRa on the Master; tier-1 *and* tier-2 on a Slave). If not, the IWDG resets it, `attempts++`, and at 3 it swaps back to the previous bank.

The previous image stays intact in the other bank, so **rollback is a swap, not a copy**. ⚠ Keep config outside the swapped banks (FRAM, or identical pages in both banks), or it moves with the swap ([[Persistence-And-Storage]]).

**MCUboot** (Apache-2.0, well-audited, `imgtool.py` signing) is the preferred way to implement this, over ST's X-CUBE-SBSFU. The external-NOR staging design (image staged in SPI NOR, copied in by a bootloader) remains a fallback if the final gateway MCU is single-bank.

## Signing

- **ECDSA P-256 + SHA-256.** The L476 has no PKA or AES hardware, so verification runs in software. That's fine: it only happens once per update. It does have a **TRNG**.
- **Verify, don't encrypt.** Authenticity is the real requirement (stopping someone turning a city's lights off); confidentiality of the binary costs debuggability.
- **The public key lives in the protected boot stage**, not in config. Enable **readout protection level 1** in production. ⚠ **Never RDP level 2**: it's irreversible and kills RMA and field debug.
- **Header:** `{magic, header_version, image_version, target_hw_id, payload_len, payload_sha256[32], signature[64], build_timestamp}`. **`target_hw_id` stops a slave image being installed on the master**, or a device image on a gateway, at 2 a.m.
- **Monotonic version check:** refuse `image_version <= current` unless a signed downgrade flag is set.

## How slow is LoRa, really?

Raw goodput at CR 4/5 with ~200 B frames:

| Config | Frame ToA | Goodput | 200 KB, pure airtime |
|---|---|---|---|
| SF7 / BW125 | 326 ms | 613 B/s | 5.6 min |
| SF8 / BW125 | 578 ms | 346 B/s | 9.9 min |
| SF9 / BW125 | 1.07 s | 187 B/s | 18 min |

Multiply by **3–5×** for fragment headers, gap repair and leaving room for normal traffic. Under the 400 ms dwell limit, ~200 B frames are only legal at SF7, so **firmware transfer runs at SF7**.

**The duty cycle is the real limit.** At ≤1% per box, a gateway may transmit only **~36 s per hour**. At SF7 that's roughly **20 KB per hour**, so a ~200 KB gateway image takes **on the order of 10 hours** of transfer per Slave. ⚠ This is an estimate from the adopted regulatory profile; it would shrink a lot if NTC's actual rules are looser. Earlier estimates of 15–30 minutes per Slave ignored the duty cycle.

### Per-pole devices

There's now one LoRa device per pole (up to 60 per Slave), on Blue Pills with 64 KB single-bank flash. **Unicast updates are impractical:** 60 devices × a ~64 KB image at ~20 KB per hour is roughly a week per section. ⚠ Estimate.

The only viable path is **multicast with forward erasure coding** (LoRa Alliance TS004-style fragmentation): the Slave broadcasts the image once to the whole cell, devices reconstruct missing fragments from the redundancy, and only report a completion bitmap. Devices would also need somewhere to stage the image (external flash), which is the device developer's hardware decision. Until then, **plan device updates by wire (SWD) or by swapping units.**

## Rules

1. **The Master gets firmware over RS485, not LoRa.** At 115200 baud a 200 KB image is ~20 s of wire time, ~60 s with framing. Implement it as a `start_ota` mailbox command plus chunked `FC16` writes ([[Modbus-Register-Map-Design]]).
2. **Never brick on a partial transfer.** The running image is untouched until a verified swap, so an aborted transfer costs time, not a truck roll.
3. **Never update all Slaves at once.** One section at a time, in a maintenance window, while the lighting state is static.
4. Chunks of ~200 bytes with a `{fragment_index, total}` header and **bitmap-based gap repair**, not per-fragment ACKs. No per-chunk CRC needed: the PHY already has one and the whole image is hashed.

## Related

- [[Custom-Protocol-Security]]
- [[Persistence-And-Storage]]
- [[Custom-Protocol-Frame-Design]]
- [[LoRa-Airtime-Budget]]
- [[Development-Roadmap-And-Gates]]

## Sources

- Research synthesis, 2026-09-09. AN5554 LoRaWAN FUOTA with CubeWL; FUOTA energy/latency analysis: https://arxiv.org/pdf/2508.10588 · dual-bank swap OTA: https://blog.carrese.eu/articles/2026-06-13-stm32-dual-bank-flash-swap-ota/ · STM32WL crypto performance: https://wiki.st.com/stm32mcu/wiki/Security:STM32WL_Series:_Cryptographic_Library_Performance

## History

- 2026-09-09: designed around the STM32WLE5's single-bank flash, with external NOR staging; LoRa updates estimated at 15–30 min per slave.
- 2026-10-14: retargeted to dual-bank STM32L4 (bank swap). Re-estimated LoRa transfer under the ≤1% per-box duty cycle, added the per-pole device problem, and recorded that OTA is outside the current contracted scope.

Back to [[Index]]
