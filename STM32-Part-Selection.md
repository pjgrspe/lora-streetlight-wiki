# STM32 Part Selection

> **Superseded 2026-10-14.** With the client specifying **SX1276** radios and **up to 4 radios per gateway**, the integrated-radio STM32WLE5 / RAK3172 recommendation no longer applies (one on-die SX126x core, wrong radio, and no way to add three more). New direction: **one common gateway board for master and slave** using a general-purpose STM32 with external SX1276 modules, populated with 2 radios (master) or 4 (slave).
> - Pin budget per radio: NSS, DIO0, DIO1, NRESET (+ optional DIO2/3) → ~16–20 GPIO for 4 radios, plus **two SPI buses** (primary and backup radio of each pair on different buses, see [[Dual-Radio-Redundancy-And-Coexistence]]), plus RS485 UART, console UART and FRAM/NOR SPI.
> - **Candidate: STM32G0B1/G0C1 (64-pin)**, the old fallback: 512 KB dual-bank flash, 144 KB RAM, 6 UARTs, 3 SPI. ⚠ **G0B1 has no hardware AES or TRNG.** AES-CMAC in software on a 64 MHz M0+ is fine at these frame rates. **G0C1** is believed to add AES + RNG; verify before choosing. **Alternative with more headroom: STM32L4 / G4 (Cortex-M4F)**. The final choice depends on what "STM boards" the client already has (see [[Open-Questions]] Q22).
> - Good news: dual-bank flash makes OTA simpler than the WL path ([[Firmware-Update-Over-LoRa]]). The RAM budget below grows only slightly (a registry of up to ~140 devices instead of 134 cache blocks — about the same).

## The decision that matters: integrated radio vs external SX126x

The **STM32WL** is the only STM32 with an on-die Semtech radio — functionally an SX126x core reached through an internal `SUBGHZSPI` peripheral with the same opcode set as the discrete part. LoRa modulation exists only on the **WLx5** parts (WL55/WL54/WLE5/WLE4).

**What it removes:** one IC, ~10 passives, the external SPI bus, and the BUSY/DIO1/NRESET/NSS routing — plus the entire SX126x SPI timing hazard class (no BUSY-before-NSS race, no missed-IRQ window across a physical bus, no board-level SPI EMI on a net adjacent to 900 MHz).

**What it does not remove:** the RF matching network (still per AN5457 — real RF layout work), the TCXO, and the RF switch.

## Recommendation

**Primary — pre-certified STM32WLE5 module: `RAK3172-T`** (STM32WLE5CC + TCXO, castellated SMD). It hands you the AN5457 matching network, the TCXO, the 32.768 kHz LSE, and shielding — plus **existing radio type-approval test reports, which shortens NTC type approval** (see [[Philippines-Regulatory-Context]]). You keep full firmware control: erase the vendor AT firmware and flash your own CubeWL-based image over SWD. Alternative: Seeed Wio-E5 / LoRa-E5 (STM32WLE5JC), but ⚠ that one is −40…+85 °C only, which is marginal for the cabinet.

**Do not use the dual-core STM32WL55.** Its M0+ exists so ST can ship an isolated MAC layer; adopting IPCC + HSEM + dual-image flashing on a mains-powered gateway buys nothing.

**Fallback — `STM32G0B1CBT6` + external SX1262** (e.g. Ebyte E22-900M22S, the SPI-transparent variant). Costs +1 IC and an SPI bus, but gives **512 KB dual-bank flash, 144 KB RAM and 6 UARTs for less money**. If the WL lead time or the single-bank flash is a blocker, this is technically the *safer* choice.

## The STM32WL constraint you must design around

**256 KB flash, single bank, 128 pages × 2 KB, ECC 8 bits per 64-bit word.**

- **There is no dual-bank swap on STM32WL.** This is the biggest architectural difference from G0B1/G4/L4/H5, and it drives the bootloader design in [[Firmware-Update-Over-LoRa]].
- ECC means **you cannot rewrite a 64-bit flash word without erasing its page** — no incremental append tricks in internal flash. See [[Persistence-And-Storage]].

Peripherals (STM32WLE5CC): 2 × USART + 1 × LPUART (all with hardware RS485 Driver-Enable), 2 × SPI, 3 × I²C, 2 × DMA × 7 channels with DMAMUX, TIM2 as 32-bit 4-channel, 3 × LPTIM, IWDG + WWDG, RTC with TAMP and backup registers, **AES-128/256, PKA, TRNG, CRC**.

**Take the 105 °C temperature grade (suffix `7`)** — a Philippine roadside cabinet in sun will exceed 70 °C internal. See [[Enclosure-Power-And-Surge]].

## Flash and RAM budget

| Item | Flash | RAM |
|---|---|---|
| Bootloader (separate, WRP-locked) | 16 KB | — |
| HAL/LL + startup | 20–30 KB | 2 KB |
| FreeRTOS + 5 tasks | 8 KB | 6–10 KB |
| SubGHz_Phy / radio driver | 15–20 KB | 1 KB |
| Custom LoRa MAC (scheduler, ARQ, frag, crypto glue) | 20–25 KB | 4 KB |
| Modbus RTU slave | 8–12 KB | 1 KB |
| **Telemetry cache, 134 devices × 32 regs** | — | **8.6 KB** |
| Ring buffers and queues | — | 4 KB |
| LittleFS + SPI-NOR driver | 12–18 KB | 4–6 KB |
| ECDSA P-256 verify + SHA-256 | 6–10 KB | 2 KB |
| Log ring, diagnostics, config | 4 KB | 3 KB |
| **Total** | **~130–165 KB** | **~36–42 KB** |

Fits 240 KB app / 64 KB RAM with 30–40% headroom. It does **not** fit two 120 KB on-chip A/B slots — hence external NOR staging.

## Family comparison, for the record

| Family | Flash/RAM (top) | Dual-bank | UARTs | Radio | Note |
|---|---|---|---|---|---|
| STM32F0/F1 | 256K/32K, 1M/96K | no/some | 2–5 | ext | Legacy; no reason to start here |
| STM32F4 | 2M/256K | yes | 4–8 | ext | Overkill power and heat |
| **STM32G0** | 512K/144K | **yes (G0Bx/Cx)** | **up to 6** | ext | ★ Best cheap alternative, ~$0.63–3 |
| STM32G4 | 512K/128K | yes | 4–5 | ext | Analog silicon you won't use |
| STM32L4 | 1M/320K | yes (L4+) | 4–5 | ext | Good if you need big RAM |
| STM32H5 / U5 | 2M/640K, 4M/2.5M | yes | 6+ | ext | Overkill; best secure-boot story |
| **STM32WLE5** | 256K/64K | **NO** | 2 + LPUART | **integrated** | ★ Primary |

## Development board recommendation (added 2026-10-14)

Requirements for one gateway board, worst case being a slave with 4 SX1276 radios:

| Need | Pins / peripherals |
|---|---|
| 4 × SX1276 (NSS, DIO0, DIO1, NRESET each) | 16 GPIO |
| 2 SPI buses: the primary and backup radio of each pair on different buses ([[Dual-Radio-Redundancy-And-Coexistence]]) | SPI1 + SPI2 |
| FRAM / NOR storage | SPI3, or share a bus |
| RS485 to the server (master): TX, RX, DE | 1 USART |
| Console | ST-LINK virtual COM port (USART2) |
| RTC with battery backup, RNG for nonces and join backoff | on-chip |

That's ~30 pins, which rules out 32-pin boards (Nucleo-32) and the Blue Pill.

| Board | MCU | Flash / RAM | Dual-bank | RNG / AES | Verdict |
|---|---|---|---|---|---|
| **NUCLEO-L476RG** | Cortex-M4F, 80 MHz | 1 MB / 128 KB | ✅ | RNG ✅ / AES ❌ | **★ Recommended.** Plenty of headroom; RNG on chip. Very widely stocked. **One of the reference boards for LoRaMac-node's SX1276 driver**, so the radio driver port starts from known-good code |
| NUCLEO-G474RE | Cortex-M4F, 170 MHz | 512 KB / 128 KB | ✅ | RNG ✅ / AES ❌ | Good alternative; more CPU than needed |
| NUCLEO-G0B1RE | Cortex-M0+, 64 MHz | 512 KB / 144 KB | ✅ | ❌ / ❌ | Cheapest that fits. No RNG, so entropy must come from somewhere else. Acceptable, but second choice |
| NUCLEO-F446RE | Cortex-M4F, 180 MHz | 512 KB / 128 KB | ❌ | RNG ✅ / AES ❌ | Works, but single-bank flash makes later OTA harder |
| Blue Pill / Black Pill | F103 / F401–F411 | small | ❌ | — | **Not for gateways**: too few SPI buses and pins for 4 radios, no on-board debugger. **The devices *are* on Blue Pill**, see below |

⚠ On Nucleo-64 boards, VBAT is tied to 3V3 by default. Check the board manual (UM1724) for the solder bridge that separates it before fitting an RTC backup cell.

**Devices are on Blue Pill (STM32F103C8)** (project owner, 2026-10-14). The gateways stay on a Nucleo-64-class board. So there are **two MCU families**, and the device-role protocol library ([[Team-And-Ownership]]) must fit the smaller one:

| Blue Pill constraint | What the device-role library must do |
|---|---|
| Cortex-M3, 72 MHz, **no FPU** | Integer-only protocol code. No float in the library. (The device's own astro fallback can use soft-float; that's the device developer's code) |
| **64 KB flash** (C8; many chips have 128 KB, but don't rely on it), **20 KB RAM** | Budget: library **≤ 16 KB flash, ≤ 3 KB RAM**, including software AES-128 (CMAC + CTR) and the SX1276 driver. No malloc |
| **No RTOS assumed** | Event/callback API driven from the device's main loop and the DIO0/DIO1 interrupts. No blocking calls |
| **No hardware RNG or AES** | Software AES. Randomness for join backoff and slot jitter from the SX1276 wideband RSSI LSB (`RegRssiWideband`), seeded with the chip UID. `DevNonce` is a persisted monotonic counter, so it needs no randomness |
| **Flash endurance ~10k erase cycles**, no FRAM | Frame-counter high-water mark written **every N frames** (e.g. 256) as an append log in one page, skipping forward on reboot. Slot assignment persisted only when it changes |
| **Counterfeit / clone chips are common** (CKS32, CS32, GD32 marked or remarked) | Bench-test the library on the *actual* boards supplied. Timing and flash behaviour differ on clones. ⚠ Raise with the device developer for anything beyond the prototype |
| No on-board debugger | Device developer needs ST-LINK/V2 clones or a V3MINIE |

The device developer's per-pole production hardware will probably be a custom board anyway. Keep the library portable C behind `radio_if.h` / `time_if.h` / `store_if.h` so it moves with them.

**Radio modules:** SX1276 **915 MHz** variants only, e.g. HopeRF **RFM95W-915** (or a breakout of it), or Ai-Thinker **Ra-01H** (803–930 MHz). **Not** Ra-01/Ra-02, which are 433 MHz SX1278. Each module needs its **own antenna connector** (u.FL/SMA), because the redundancy depends on independent antennas.

## ⚠ Procurement risk

**STM32WL55CCU6 was quoted at ~$9.43/1 with a 30-week factory lead time beyond stock.** That lead time — not the price — is the real risk. WLE5CCU6 is typically meaningfully cheaper (dual-core parts carry a premium) but a live quote was not obtainable. **Get a current quote and lead time before committing, and consider a buffer buy or the G0B1 fallback.** Tracked in [[Open-Questions]].

## Related

- [[Radio-Hardware-Selection]]
- [[Firmware-Architecture]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. STM32WLE5CC: https://www.st.com/en/microcontrollers-microprocessors/stm32wle5cc.html · AN5457 RF matching: https://www.st.com/resource/en/application_note/an5457-rf-matching-network-design-guide-for-stm32wl-series-stmicroelectronics.pdf · flash organisation from the ST STM32WL5 FLASH training deck · RAK3172: https://docs.rakwireless.com/product-categories/wisduo/rak3172-module/overview/
