# Firmware Update Over LoRa

> **Updated 2026-10-14.** The STM32WL single-bank constraint driving the NOR-staging design below **probably no longer applies**. The gateways are moving to an STM32 with external SX1276 radios (likely G0B1/G0C1 with dual-bank flash; see [[STM32-Part-Selection]]), which makes the "materially simpler" dual-bank path available. Per-pole devices bring back the 128-node FUOTA problem described below. **OTA of any kind is excluded from the v5 base quote**: gateways are updated over SWD/RS485 at prototype stage.

## The STM32WL constraint and the way around it

**STM32WL flash is 256 KB, single bank — there is no `SWAP_BANK`.** ST's own FUOTA approach partitions the single bank into bootloader + active slot + download slot, which on 256 KB leaves both slots cramped.

**Recommended architecture: external-NOR staging plus in-place install.**

```
Internal flash (256 KB)              External SPI NOR (8 MB)
  0x0800_0000  BOOTLOADER 16 KB        0x00_0000  slot_staging  512 KB
               (WRP-locked, immutable) 0x08_0000  slot_golden   512 KB
  0x0800_4000  APPLICATION 236 KB      0x10_0000  LittleFS      7 MB
  0x0803_F000  CONFIG page A 2 KB
  0x0803_F800  CONFIG page B 2 KB
```

Flow:

1. App receives the image into `slot_staging`, chunk by chunk, each chunk CRC-checked.
2. App verifies the whole staged image: **SHA-256 + ECDSA-P256 signature**.
3. App writes an "install pending" record to FRAM (`target=staging, attempts=0`) and reboots.
4. **Bootloader re-verifies the signature itself** — never trust the app's verdict — copies staging → internal app region, verifies the SHA-256 **read back from internal flash**, clears the pending record, boots.
5. App must call `mark_healthy()` within ~10 minutes, gated on **both a successful Modbus exchange and a successful LoRa round trip**. If not, IWDG resets, `attempts++`, and at 3 the bootloader installs `slot_golden` instead. On success the current image is promoted to golden.

Why this beats squeezing A/B into 256 KB: the app can grow to 236 KB, staging is effectively unlimited, and the golden image is out of reach of an app bug. Cost: one $0.70 NOR chip.

**If you take the STM32G0B1 fallback** ([[STM32-Part-Selection]]) you get true dual-bank swap and the copy step disappears — materially simpler, and the strongest technical argument for that part.

## Signing

- **ECDSA P-256 + SHA-256.** The STM32WL has hardware **PKA, AES and TRNG** on die.
- **Verify, don't encrypt** — authenticity and integrity are the real requirement (stopping someone turning a city's lights off). Confidentiality of the binary is secondary and costs you debuggability.
- **Public key lives in the bootloader's WRP-protected pages**, not in config. Enable **RDP Level 1** in production. ⚠ **Never RDP Level 2** — irreversible, and it kills RMA and field debug.
- **Header:** `{magic, header_version, image_version, target_hw_id, payload_len, payload_sha256[32], signature[64], build_timestamp}`. **Include `target_hw_id`** — this system has four different firmware images (master, slave, meter, driver) sharing one radio network, and that field is what prevents installing a slave image on the master at 2 a.m.
- **Monotonic version check:** refuse `image_version <= current` unless a signed downgrade-authorised flag is set. Blocks rollback-to-vulnerable attacks.
- **MCUboot** (Apache-2.0, well-audited, `imgtool.py` signing) is preferable to X-CUBE-SBSFU for a project this size — better documented and not tied to ST's licence.

## Is OTA over LoRa actually feasible?

Computed goodput at CR 4/5, 200 B frames:

| Config | Frame ToA | Goodput | 200 KB image, pure airtime |
|---|---|---|---|
| SF7 / BW125 | 326 ms | 613 B/s | **5.6 min** |
| SF8 / BW125 | 578 ms | 346 B/s | 9.9 min |
| SF9 / BW125 | 1.07 s | 187 B/s | 18 min |
| **SF7 / BW500** | 81 ms | **2450 B/s** | **1.4 min** |
| SF12 / BW125 | ~7.0 s | 28 B/s | **2.0 hours** |

Multiply by **3–5x** for fragment headers, ACK/NACK round trips, retransmission, and leaving channel capacity for normal telemetry.

> **Verdict: OTA to the 3 slave gateways is entirely feasible — budget 15–30 minutes per slave at SF7/BW125.** Do it at 2 a.m. when the lighting state is static.

**But pushing firmware to 128 luminaires over the second LoRa hop is a different problem:** at ~600 B/s shared, 128 × 200 KB serially is **~12 days**. You would need multicast with forward erasure coding, not unicast. Since our 128 luminaires are behind **wired DALI**, this problem does not arise — which is a strong argument for the two-tier design. See [[LoRa-For-Control-Critique]].

## Rules

1. **The master gets firmware over RS485, not LoRa.** At 115200 baud a 200 KB image is ~20 s of wire time, ~60 s with framing and ACKs. Implement it as a `start_ota` mailbox command plus chunked `FC16` writes, or YMODEM on a secondary UART.
2. **Dwell time, not throughput, is the binding constraint.** Under a 400 ms dwell limit, SF9 allows only ~11 bytes of payload and SF12 allows less than a preamble — **so FUOTA must run at SF7/SF8**, possibly at BW250/500. See [[Philippines-Regulatory-Context]].
3. **Never brick on a partial transfer.** Because the image goes to NOR first and the running app is untouched until the bootloader installs, an aborted 20-minute transfer costs 20 minutes, not a truck roll.
4. **Never update all 3 slaves at once.** One section at a time, daytime maintenance windows only.
5. Chunk size ~200 bytes with a 4-byte `{fragment_index, total}` header. Don't add a per-fragment CRC — the PHY already has one and the whole image is SHA-256'd.

## Related

- [[Custom-Protocol-Security]]
- [[Persistence-And-Storage]]
- [[Development-Roadmap-And-Gates]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. AN5554 LoRaWAN FUOTA with CubeWL; FUOTA energy/latency analysis: https://arxiv.org/pdf/2508.10588 · dual-bank swap OTA: https://blog.carrese.eu/articles/2026-06-13-stm32-dual-bank-flash-swap-ota/ · STM32WL crypto performance: https://wiki.st.com/stm32mcu/wiki/Security:STM32WL_Series:_Cryptographic_Library_Performance
