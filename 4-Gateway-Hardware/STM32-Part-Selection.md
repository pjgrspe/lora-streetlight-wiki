# STM32 Part Selection

Which STM32 runs the gateways and which runs the devices. The gateways use **one common Nucleo-64-class board (recommended: NUCLEO-L476RG)** with external SX1276 radios. The devices use **Blue Pill (STM32F103C8)**, which constrains the shared protocol library.

## Key points

- **One common gateway board design.** The Master populates 2 radios, each Slave 4. The hardware itself is built by a separate hardware developer, following the client-supplied STM32 dev boards + SX1276 modules and pgorospe's gateway hardware requirements spec.
- **Recommended gateway board: NUCLEO-L476RG.** It has 1 MB dual-bank flash, 128 KB RAM, 3 SPI buses, an on-chip RNG, and is a LoRaMac-node SX1276 reference board. Alternatives are G474RE, G0B1RE and F446RE.
- **The Blue Pill is not for gateways.** It has too few SPI buses and pins for 4 radios, and no on-board debugger.
- **The devices are Blue Pill STM32F103C8 + SX1276** (another developer's firmware). The device-role protocol library must fit ≤16 KB flash / ≤3 KB RAM, bare-metal, with no FPU.
- **Redundant radio pairs share nothing:** separate SPI bus, reset line, LDO/fused rail, antenna and coax.
- **Three SPI buses are needed:** SPI1 and SPI2 split each radio pair, and SPI3 carries the FRAM and NOR. The proposed NUCLEO-L476RG pin map is in spec 06 §3 (v0.1), every pin still ⚠ to verify.

## Gateway board requirements

Worst case is a Slave with 4 SX1276 radios:

| Need | Pins / peripherals |
|---|---|
| 4 × SX1276 (NSS, DIO0, DIO1, NRESET each; optional DIO2/3) | ~16–20 GPIO |
| 2 SPI buses: primary and backup radio of each pair on different buses ([[Dual-Radio-Redundancy-And-Coexistence]]) | SPI1 + SPI2 |
| FRAM / NOR storage | **SPI3** (never on a radio bus) |
| RS485 to the server (Master): TX, RX, DE | USART1 with hardware DE |
| Console | ST-LINK virtual COM port (USART2) |
| RTC with battery backup; RNG for nonces and join backoff | on-chip |

That is ~30 pins, which rules out 32-pin boards (Nucleo-32) and the Blue Pill.

### Proposed allocation on NUCLEO-L476RG

Specified in `Specs/specs/06-gateway-hardware-requirements.md` §3 (v0.1). In short:

| Function | Pins |
|---|---|
| SPI1: primary radios (Master R1; Slave U1 + D1) | PA5 / PA6 / PA7 |
| SPI2: backup radios (Master R2; Slave U2 + D2) | PB13 / PB14 / PB15 |
| SPI3: FRAM + NOR | PC10 / PC11 / PC12 |
| RS485 (Master) | USART1: PA9 TX, PA10 RX, PA12 DE |
| Console | USART2 on the ST-LINK virtual COM port, PA2 / PA3 (reserved) |
| Radio DIO0/DIO1 | PC0–PC3, PB4, PC5–PC7: eight distinct EXTI lines |

- ⚠ **PA5 also drives the user LED LD2** through solder bridge SB21. Open SB21 on the bench boards so the LED doesn't load the SPI1 clock.
- ⚠ PA11/PA12 are also USB pins; confirm nothing is fitted on them.
- ⚠ Every pin must be checked against UM1724 and the L476RG datasheet before wiring (spec 06 O-06-1).

**Bench set from week 2** (SOW assumption A7, spec 06 §12): 3 NUCLEO-L476RG (1 Master + 2 Slaves), 10 SX1276 915 MHz modules (2 + 4 + 4), 3 FRAM/NOR breakouts, an isolated USB-RS485 adapter; from M4, 1 meter + 6 LED devices from the device developer.

## Gateway board options

| Board | MCU | Flash / RAM | Dual-bank | RNG / AES | Verdict |
|---|---|---|---|---|---|
| **NUCLEO-L476RG** | Cortex-M4F, 80 MHz | 1 MB / 128 KB | ✅ | RNG ✅ / AES ❌ | **★ Recommended.** Plenty of headroom; widely stocked. **A reference board for LoRaMac-node's SX1276 driver**, so the radio port starts from known-good code |
| NUCLEO-G474RE | Cortex-M4F, 170 MHz | 512 KB / 128 KB | ✅ | RNG ✅ / AES ❌ | Good alternative; more CPU than needed |
| NUCLEO-G0B1RE | Cortex-M0+, 64 MHz | 512 KB / 144 KB | ✅ | ❌ / ❌ | Cheapest that fits. No RNG, so entropy must come from elsewhere. Second choice. ⚠ G0C1 is believed to add AES + RNG; verify |
| NUCLEO-F446RE | Cortex-M4F, 180 MHz | 512 KB / 128 KB | ❌ | RNG ✅ / AES ❌ | Works, but single-bank flash makes later OTA harder |
| Blue Pill / Black Pill | F103 / F401–F411 | small | ❌ | — | **Not for gateways.** Used for the devices, see below |

- **No hardware AES** on any of these. AES-CMAC/CTR runs in software, which is fine at these frame rates ([[Custom-Protocol-Security]]).
- **Dual-bank flash** (L476, G474, G0B1) makes a later firmware update simpler than single-bank parts ([[Firmware-Update-Over-LoRa]]).
- ⚠ On Nucleo-64 boards, VBAT is tied to 3V3 by default. Check the board manual (UM1724) for the solder bridge that separates it before fitting an RTC backup cell ([[Time-Distribution-And-Clock-Trust]]).
- **Take a 105 °C-grade part** (suffix `7`) for the production board. A roadside cabinet exceeds 70 °C internally ([[Enclosure-Power-And-Surge]]).

## Flash and RAM budget (gateway)

Earlier estimates put the firmware at **~130–165 KB flash and ~36–42 KB RAM**: RTOS + 5 tasks, radio driver, MAC, Modbus RTU slave, a telemetry cache of ~8.6 KB for ~134 devices × 32 registers, LittleFS, logs and diagnostics. The L476's 1 MB / 128 KB leaves large headroom, including room for two on-chip firmware slots. The cache grows with the device count, which is TBC; at ~64 B per device it stays small.

## Device platform: Blue Pill (STM32F103C8)

The device developer builds the energy meter and the per-pole LED driver/monitoring device on Blue Pill + SX1276 (one radio each). There are therefore **two MCU families**, and the device-role library ([[Team-And-Ownership]]) must fit the smaller one:

| Blue Pill constraint | What the device-role library must do |
|---|---|
| Cortex-M3, 72 MHz, **no FPU** | Integer-only protocol code. No float in the library. (The device's own astro fallback can use soft-float; that's the device developer's code) |
| **64 KB flash** (C8; many chips have 128 KB, but don't rely on it), **20 KB RAM** | Budget: library **≤ 16 KB flash, ≤ 3 KB RAM**, including software AES-128 (CMAC + CTR) and the SX1276 driver. No malloc |
| **No RTOS assumed** | Event/callback API driven from the device's main loop and the DIO0/DIO1 interrupts. No blocking calls |
| **No hardware RNG or AES** | Software AES. Randomness for join backoff and slot jitter from the SX1276 wideband RSSI LSB (`RegRssiWideband`), seeded with the chip UID. `DevNonce` is a persisted monotonic counter, so it needs no randomness |
| **Flash endurance ~10k erase cycles**, no FRAM | Frame-counter high-water mark written **every N frames** (e.g. 256) as an append log in one page, skipping forward on reboot. Slot assignment persisted only when it changes |
| **Counterfeit / clone chips are common** (CKS32, CS32, GD32 marked or remarked) | Bench-test the library on the *actual* boards supplied. Timing and flash behaviour differ on clones. ⚠ Raise with the device developer for anything beyond the prototype |
| No on-board debugger | Device developer needs ST-LINK/V2 clones or a V3MINIE |

**Bench data (2026-10-28):** Blue Pills are also sold with the **STM32F103C6** (32 KB flash, 10 KB RAM); the device developer's bench board is one. Specify the C8 and check the chip marking. The device-role library at `1d97ae8` measures **≈8.5 KB** with arm-none-eabi-gcc `-Os` on the Cortex-M3 (tier-1 formats excluded), so the 16 KB budget holds with an SX1276 driver. See [[DALI-Bench-Rig-And-Results]].

The device developer's per-pole production hardware will probably be a custom board. Keep the library portable C behind `radio_if.h` / `time_if.h` / `store_if.h` so it moves with them.

## Radio modules

SX1276 **915 MHz** variants only: HopeRF **RFM95W-915** (or a breakout of it), or Ai-Thinker **Ra-01H** (803–930 MHz). **Not** Ra-01/Ra-02, which are 433 MHz SX1278 parts. Each module needs its **own antenna connector** (u.FL/SMA), because the redundancy depends on independent antennas. Details in [[Radio-Hardware-Selection]].

## Alternatives considered

| Option | Why it is not the design |
|---|---|
| **STM32WLE5 / RAK3172-T** (STM32 with on-die SX126x radio, pre-certified module) | The earlier primary recommendation. It carries one integrated SX126x radio, so it can't be the client's SX1276 and can't host 4 radios. It also has single-bank 256 KB flash. Its main remaining merit was a pre-certified module for NTC type approval |
| STM32WL55 (dual-core) | Same radio limits plus IPCC/dual-image complexity; no benefit here |
| STM32F0/F1 class for gateways | Too little RAM, flash and SPI for 4 radios |
| STM32H5 / U5 | Overkill; best secure-boot story if ever needed |

⚠ The STM32WL55 line was quoted at ~$9.43/1 with a 30-week factory lead time. That's no longer on the critical path, but **lead time, not price, is the usual procurement risk**: get live stock and lead time for the chosen Nucleo boards before committing.

## Related

- [[Radio-Hardware-Selection]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Firmware-Architecture]]
- [[Team-And-Ownership]]

## Sources

- Research synthesis, 2026-09-09. STM32WLE5CC: https://www.st.com/en/microcontrollers-microprocessors/stm32wle5cc.html · AN5457 RF matching: https://www.st.com/resource/en/application_note/an5457-rf-matching-network-design-guide-for-stm32wl-series-stmicroelectronics.pdf · flash organisation from the ST STM32WL5 FLASH training deck · RAK3172: https://docs.rakwireless.com/product-categories/wisduo/rak3172-module/overview/
- Project owner, 2026-10-14: client supplies STM32 dev boards + SX1276 modules; devices on Blue Pill; separate gateway hardware developer.
- `Specs/specs/06-gateway-hardware-requirements.md` (v0.1, 2026-10-21).

## History

- 2026-09-09: STM32WLE5 / RAK3172-T recommended, with STM32G0B1 + SX1262 as fallback.
- 2026-10-14: Client specified SX1276 and up to 4 radios per gateway, so moved to a common STM32 + external SX1276 board. NUCLEO-L476RG recommended. Devices confirmed on Blue Pill; device-constraint table added.
- 2026-10-21: aligned with spec pack v0.1 (third SPI bus required, proposed pin map, SB21/PA5 note, bench set).
- 2026-10-28: device platform: F103C6 Blue Pills exist; library measured at ≈8.5 KB on the Cortex-M3 ([[DALI-Bench-Rig-And-Results]]).

Back to [[Index]]
