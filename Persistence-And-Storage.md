# Persistence And Storage

## What lives where

| Data | Rate of change | Medium | Why |
|---|---|---|---|
| Bootloader + app image | rare | **internal flash** | Execute-in-place required |
| Config: unit ID, baud, RF params, lat/long, schedules, device roster | commissioning | **internal flash, 2-page ping-pong** | No extra parts |
| Energy totalizers, burn hours, boot count, health counters | every few seconds | **FRAM** | 10¹² endurance, byte-writable, no erase, no wear-levelling |
| Event/fault log, telemetry history | continuous | **SPI NOR + LittleFS** | Cheap capacity |
| OTA staging image | per update | **SPI NOR, raw sectors** | Solves the single-bank problem |
| Live telemetry cache (134 × 32 B) | continuously | **RAM only** | It is a *cache*; regenerate from downstream on boot |

## Internal flash for config — the ECC trap

STM32WL flash is 128 pages × 2 KB, single bank, **ECC over each 64-bit word**. Consequences:

- The minimum programming unit is a **64-bit doubleword**, writable **once per erase cycle**. Writing a second time to the same address — even 1→0 — causes `PROGERR` or, worse, a **latent ECC double-error that faults on read**.
- Endurance is typically 10 k cycles. ⚠ Verify for the exact part.

**Correct pattern — two-page ping-pong journal:**

```
Page A and Page B (2 KB each), each holding N slots of 64 B:
  slot = { uint32 seq; uint32 len; uint8 payload[52]; uint32 crc32; }

Write: append to the next erased slot in the active page, seq++
Read:  scan both pages, take the highest valid seq with a good CRC32
Full:  erase the OTHER page, write slot 0 there, and only after that
       succeeds, erase the old page
```

This gives power-loss safety (a torn write leaves a bad CRC and is ignored) and multiplies endurance by the slot count: 2 × 32 × 10 k = **640 k config writes**.

> **Never hold config in a single page you erase-then-write.** That is a 4 ms window in which a brownout destroys your commissioning data.

Store the [[Device-Identity-Namespaces]] registry the same way — journal plus two-copy commit with CRC. **A registry corrupted by a brownout orphans the whole network**, and a Philippine street-lighting cabinet will brown out.

## External storage

- **FRAM: `MB85RS256B`** (256 kbit, SPI, **10¹² cycles, no erase**, ~$2–3). This is the correct part for a kWh totalizer updated every 5 s: NOR would burn its endurance and internal flash physically cannot do it.
- **SPI NOR: `W25Q64JV`** (8 MB, 4 KB sectors, ~100 k cycles/sector, ~$0.70). ⚠ **Check the temperature grade** — take the industrial variant.

Both on one SPI bus with separate CS lines. Total ~$3.50.

**If you must cut one:** keep FRAM, put logs in a RAM ring dumped over Modbus/RTT, and use the internal-flash ping-pong for config. Under severe BOM pressure: keep NOR only and journal the totalizers there via LittleFS, accepting a once-per-minute update rate instead of every 5 s.

## LittleFS

The right filesystem: **copy-on-write with real power-loss guarantees** — it falls back to the last known-good state on an interrupted write — plus dynamic wear-levelling. Designed for exactly this target class.

- RAM cost is `read_size + prog_size + 2×cache_size + lookahead_size/8`. With `cache_size = 256` and `lookahead_size = 512` you are at ~1 KB.
- **Set `block_cycles = 500`** so LittleFS relocates metadata blocks and actually exercises wear-levelling. ⚠ **`block_cycles = -1` disables wear-levelling entirely** — a common misconfiguration.
- Use **append-only, size-capped, rotating log files** (`log.0`…`log.7`), not one growing file. Call `lfs_file_sync()` on a boundary — every 16 entries or 60 s — not per entry.
- **Reserve a raw NOR region outside LittleFS for the OTA staging image.** Streaming a 200 KB image through a filesystem adds failure modes for no benefit; write it to raw sectors with its own header and CRC.

A hand-rolled circular log over raw NOR with per-sector sequence headers is a defensible alternative for a device whose only files are `config`, `log` and `image` — it saves ~15 KB of flash. **LittleFS is still the better trade**: the power-loss guarantees are worth more than the flash.

## Sizing the outage buffer

⚠ Unresolved: how many events per day, and how long an outage must you survive? If the answer is "7 days of full telemetry", an internal-flash ring will not do. Rough figure from the slave side: 2 devices × 1 reading/10 s × 24 h × 32 B ≈ **550 kB** — too much for on-chip flash, hence external NOR. **Better answer: downsample.** Keep 15-minute summaries plus full resolution for the last hour. Tracked in [[Open-Questions]].

## Related

- [[Firmware-Update-Over-LoRa]]
- [[Watchdog-And-Fault-Recovery]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. LittleFS design: https://github.com/littlefs-project/littlefs/blob/master/DESIGN.md · STM32WL flash organisation from the ST STM32WL5 FLASH training deck
