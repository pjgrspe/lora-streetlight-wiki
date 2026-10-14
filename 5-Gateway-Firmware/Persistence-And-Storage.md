# Persistence And Storage

What the gateway stores, where, and how it survives a power cut mid-write. The bottom line: config in a two-page journal in internal flash, fast-changing counters in FRAM, logs and history in SPI NOR with LittleFS, and the live telemetry cache in RAM only.

## Key points

- **Never erase-then-write a single config page.** Use a two-page ping-pong journal with CRC'd slots.
- **STM32L4 flash has ECC and programs in 64-bit double-words, once per erase.** Rewriting a programmed double-word is a fault, not a correction.
- **FRAM for counters that change every few seconds** (kWh totalizers, burn hours). Flash and NOR can't take that endurance.
- **SPI NOR + LittleFS** for event logs and the outage history buffer, with `block_cycles` set so wear-levelling is on.
- **⚠ With dual-bank OTA, keep config out of the swapped banks**, or it moves with the bank swap.

## What lives where

| Data | Rate of change | Medium | Why |
|---|---|---|---|
| Bootloader + app image | rare | **internal flash** | Execute in place |
| Config: unit ID, RF channel plan, lat/long, schedules, device registry, slot table (Slave) | commissioning / on change | **internal flash, 2-page ping-pong** (or FRAM, see the bank-swap trap) | No extra parts |
| Energy totalizers, burn hours, boot count, health counters, frame-counter high-water marks | every few seconds | **FRAM** | 10¹² endurance, byte-writable, no erase |
| Event/fault log, outage history buffer | continuous | **SPI NOR + LittleFS** | Cheap capacity |
| OTA staging image (if used) | per update | **SPI NOR, raw sectors**, optional with dual-bank flash | See [[Firmware-Update-Over-LoRa]] |
| Live telemetry cache | continuous | **RAM only** | It's a cache; it rebuilds from downstream reports after boot |

## Internal flash for config: the ECC trap

On the STM32L4 (L476), flash programs in **64-bit double-words** with ECC on each one. ⚠ Verify the page size (expected 2 KB) and endurance (typically 10 k cycles) for the exact part.

- A double-word can be written **once per erase cycle**. Writing it again, even to clear more bits, causes a programming error or, worse, a **latent ECC error that faults later on read**.
- So there are no "append into the same word" tricks in internal flash.

**Correct pattern: two-page ping-pong journal.**

```
Page A and Page B (2 KB each), each holding N slots of 64 B:
  slot = { uint32 seq; uint32 len; uint8 payload[52]; uint32 crc32; }

Write: append to the next erased slot in the active page, seq++
Read:  scan both pages, take the highest valid seq with a good CRC32
Full:  erase the OTHER page, write slot 0 there, and only after that
       succeeds, erase the old page
```

A torn write leaves a bad CRC and is ignored, and endurance multiplies by the slot count: 2 × 32 × 10 k ≈ **640 k config writes**.

> **Never hold config in a single page you erase-then-write.** That's a window of milliseconds in which a brownout destroys the commissioning data.

Store the device registry ([[Device-Identity-Namespaces]]) and, on a Slave, the **device slot-assignment table** the same way. **A registry corrupted by a brownout orphans the network**, and a Philippine roadside cabinet will brown out. The Slave's persisted slot table is also what lets ~60 devices resume after a mains restore without all re-joining at once ([[Slave-Cell-Capacity-And-Scheduling]]).

### ⚠ The bank-swap trap

With dual-bank OTA on the L476, a bank swap remaps *both* banks. Config stored at a fixed address inside a bank is physically a different page after the swap. Either:
- keep config in **FRAM** (simplest), or
- reserve identical config pages at the same offset in both banks and copy across before swapping.

Decide this together with the update design in [[Firmware-Update-Over-LoRa]].

## External storage

- **FRAM: `MB85RS256B`** (256 kbit, SPI, **10¹² cycles, no erase**). The correct part for a totalizer written every 5 s and for frame-counter high-water marks ([[Custom-Protocol-Security]]).
- **SPI NOR: `W25Q64JV`** (8 MB, 4 KB sectors, ~100 k cycles per sector). ⚠ **Take the industrial temperature grade.**

Both share one SPI bus with separate chip selects. On a Slave, that bus must not be one of the two radio buses, or a storage fault can take out a radio pair ([[Dual-Radio-Redundancy-And-Coexistence]]). ⚠ This depends on the board having a third SPI bus, or on sharing carefully. Settle it in the hardware requirements.

**If one must go:** keep FRAM, keep logs in a RAM ring dumped over the console or Modbus, and use the internal-flash journal for config. Under severe BOM pressure, keep NOR only and journal totalizers through LittleFS at a once-per-minute rate.

## LittleFS

The right filesystem here: **copy-on-write with real power-loss guarantees** (an interrupted write falls back to the last good state) plus dynamic wear-levelling.

- RAM cost is `read_size + prog_size + 2×cache_size + lookahead_size/8`. With `cache_size = 256` and `lookahead_size = 512` that's ~1 KB.
- **Set `block_cycles = 500`** so metadata blocks get relocated. ⚠ **`block_cycles = -1` disables wear-levelling entirely**, a common misconfiguration.
- Use **append-only, size-capped, rotating log files** (`log.0`…`log.7`), not one growing file. Call `lfs_file_sync()` on a boundary (every 16 entries or 60 s), not per entry.
- If NOR is used for OTA staging, **reserve a raw region outside LittleFS** for it, with its own header and CRC.

## Sizing the outage buffer

⚠ Still open: how much history must a gateway hold while the server or the master link is down, and for how long? On a Slave with ~54 devices reporting every 60 s at ~32 B per report, that's about **2.5 MB per day**. That's far beyond internal flash, comfortable in 8 MB NOR for a couple of days.

**Better answer: downsample.** Keep full resolution for the last hour and 15-minute summaries beyond that. Tracked in [[Open-Questions]].

## Related

- [[Firmware-Update-Over-LoRa]]
- [[Watchdog-And-Fault-Recovery]]
- [[Firmware-Architecture]]
- [[STM32-Part-Selection]]

## Sources

- Research synthesis, 2026-09-09. LittleFS design: https://github.com/littlefs-project/littlefs/blob/master/DESIGN.md · STM32WL flash organisation from the ST STM32WL5 FLASH training deck (the double-word/ECC behaviour is the same class on STM32L4; ⚠ verify against the L476 reference manual).

## History

- 2026-09-09: written for the STM32WLE5 (256 KB single-bank flash).
- 2026-10-14: retargeted to STM32L4 (L476, dual-bank). Added the bank-swap config trap, the Slave slot-table persistence, and outage sizing for ~54 devices per Slave.

Back to [[Index]]
