# DALI Bench Rig And Results

On 2026-10-28 the device developer's bench drove a real DALI-2 LED driver from a Blue Pill, two-way, through an off-the-shelf opto-isolated interface. This note records the rig, its wiring and part specifications, what was measured, and what it changes for the per-pole device.

## Key points

- **Two-way DALI-2 works from a Blue Pill** with small bare-register firmware: TX paced by a timer, RX on pin-change interrupts with 1 MHz timestamps. Loopback half-bits measured **357–388 µs low and 445–476 µs high**, inside DALI-2's 333–500 µs window.
- **The bench driver answers as DT6, DALI-2** (version 0x08), with Power On and System Failure Level both 254 and a fade time set. **Its GTIN is blank**, and it has **no bus supply**: the device must provide one.
- **The Waveshare Pico-DALI2's TX is active-high** (pin HIGH pulls the bus low), the opposite of what its vendor demo suggests, and the board **lets go of the bus within 10 ms** on its own.
- ⚠ **Fail-safe layer 4 has a hole on this kind of hardware.** If the bus supply outlives the MCU and the interface releases the bus by itself, a dead MCU never interrupts the bus, so the driver keeps its last level instead of going to System Failure Level. Raised as spec 05 O-05-7 and [[Open-Questions]] Q35.
- The bench Blue Pill is an **STM32F103C6** (32 KB flash, 10 KB RAM), not a C8. The device-role library measures **≈8.5 KB** on the Cortex-M3, inside its 16 KB budget.

## Bench rig

| Part | Specification | Role |
| --- | --- | --- |
| **LTECH SE-12-350-700-W1DS** | DALI-2, IEC 62386-207 **DT6**. Constant current 350–700 mA set by DIP switches (0 0 0 = 350 mA), ≤35 V, 0.7–8.4 W, 115/230 VAC. Draws ~1.6 mA from the bus, supplies none. DALI Alliance listing: firmware 2.0, hardware 1.0, GTIN 6971379119911, "D4i certified" but parts 250–253 not listed | Driver under test |
| **Waveshare Pico-DALI2** (SKU 28643) | Bridge rectifier (polarity-free), 1 A fuse, two EL1018 optocouplers (isolated both ways), 2N60 MOSFET pulls the bus low. Logic powered from VSYS (5 V). No bus supply | DALI interface |
| **Blue Pill, STM32F103C6T6, USB-C** | Cortex-M3, 72 MHz from the 8 MHz crystal, **32 KB flash, 10 KB RAM**. Its USB-C port has no CC resistors: it gets power only from a USB-A to USB-C cable | Controller |
| **ATX power supply, 12 V rail, through 150 Ω 2 W** | ~12 V idle; ~80 mA (≈1 W in the resistor) while the bus is pulled low | Bus supply |
| **FTDI FT232R USB-UART** | USART1 at 115200 baud. Also flashes the chip through its ROM bootloader (BOOT0 = 1), since no ST-LINK was at hand | Console |

## Wiring

| From | To | Signal |
| --- | --- | --- |
| Blue Pill PA8 | Pico-DALI2 header pin 9 (Chx-TX) | DALI transmit |
| Pico-DALI2 header pin 5 (Chx-RX) | Blue Pill PA0 | DALI receive. PA0 is not 5 V-tolerant: RX must stay ≤ 3.3 V (PB6 is the 5 V-tolerant alternative) |
| Blue Pill 5V / 3.3 / G | Pico-DALI2 pin 39 (VSYS) / 36 (3V3) / 38 (GND) | Interface logic power |
| ATX +12 V → 150 Ω | Pico-DALI2 DA1 and driver DA | Bus + |
| ATX GND | Pico-DALI2 DA2 and driver DA | Bus − (the DALI terminals are polarity-free) |
| FTDI RX / TX / GND | Blue Pill PA9 / PA10 / G | Console and flashing |
| Driver LED+ / LED− | LED module or dummy load | Output (empty on the bench so far) |

## Firmware under test

Bare-register C99, no vendor HAL, about 13 KB of flash and 0.4 KB of RAM:

- **TX:** the frame is encoded to a list of half-bit levels; TIM3 interrupts every 416.67 µs and sets PA8 to the next one, then releases the bus.
- **RX:** PA0 interrupts on both edges and stamps each from TIM2 counting at 1 MHz. The decoder accepts 250–600 µs as one half-bit and 650–1150 µs as two, wider than DALI-2's 333–500 µs to absorb the optocouplers' skew.
- **Loopback:** the interface hears its own transmission, so the controller decodes its own frames. That checks bus supply, TX, RX and decoding with no driver at all.

Code and a demo guide: repo `lrimando-dev/lora-streetlight-node`, branch `bench/dali-demo`, folder `bench/dali-demo/` and `docs/DALI2-DEMO-GUIDE.md`.

## Measured on 2026-10-28

| Test | Result |
| --- | --- |
| Bus idle | High |
| Loopback, 5 broadcast queries | All decoded. Half-bit 357–388 µs while low, 445–476 µs while high |
| `QUERY CONTROL GEAR PRESENT` | 0xFF (yes) |
| `QUERY DEVICE TYPE` | 6 (LED control gear) |
| `QUERY VERSION NUMBER` | 0x08 (DALI-2, IEC 62386 version 2.0) |
| `QUERY STATUS` | 0xC4 after power-up (lamp on, no short address, power cycle seen); 0x44 once commanded |
| Actual / min / max level | 254 / 1 / 254 |
| Power On / System Failure Level | 254 / 254 (as delivered; the bench never wrote them) |
| Memory bank 0 | Last location 0x1A, last bank 1. **GTIN all 0xFF (not programmed).** Firmware 2.0, hardware 1.0, identification number `BC9AE12F7CCA0956`, Part 101 and 102 version 2.0, one logical control gear |
| DAPC 0, 254, then a 13-step sweep 254 → 1 → 254 | All accepted. Actual level read 80 right after the final DAPC 254 and 254 two seconds later: the driver applies a fade time (index not yet read) |
| Driver output with no load | 30.6 V open circuit at any level. LED current not yet measured |

## Findings

| Finding | Evidence | What it means for the device |
| --- | --- | --- |
| **Pico-DALI2 TX is active-high:** pin HIGH pulls the bus low | A per-half-bit trace showed every half-bit heard as the inverse of what was sent; a static test confirmed it | Opposite of Waveshare's demo. Verify the interface's TX polarity by loopback on every new board design: a reversed setting still reads "bus high" at idle (next row) and shows only as inverted frames that no driver answers |
| **The interface releases the bus within 10 ms** on a steady TX | Holding TX at the pull level: bus low at 100–400 µs, high again by 10 ms. ⚠ Cause not traced in the schematic | A stuck TX can't take the bus down, which protects other users of it. But the bench cannot hold the bus low for the ~550 ms that triggers System Failure Level: test that by cutting the bus supply instead |
| ⚠ **A dead MCU may not trip System Failure Level** | Inference from the row above, not yet tested | Layer 4 of [[Fail-Safe-Lighting-Rule]] assumes a dead device interrupts the bus. With an always-on bus supply and a self-releasing interface it doesn't, and the driver holds its last level, possibly off or dimmed at night. The device board needs a bus supply that drops, or a bus that is pulled low, when the MCU stops (e.g. supply enabled by a watchdog-refreshed signal). Spec 05 O-05-7 |
| **The interface's RX output is weakly driven** | Enabling the MCU's internal pull-up moved RX edges by ~100 µs | Leave the RX pin floating with this board; size the RX pull-up deliberately on the device PCB |
| **12 V bus supply is marginal** | 12 V − 1.6 mA × 150 Ω ≈ 11.76 V at the driver; DALI logic high starts at 11.5 V | Enough for one driver on the bench, not for more. The device's own bus supply should sit near the nominal 16 V, current-limited ≤ 250 mA |
| **The driver has no bus supply** | Bus dead without the external 12 V | No Part 250 supply in this driver, so the device must power the bus (see [[DALI-Telemetry-And-D4i]]) |
| **The GTIN is blank** | Bank 0 bytes 0x03–0x08 read 0xFF | Identify drivers by the bank-0 identification number, not the GTIN ([[Open-Questions]] Q34) |
| **Blue Pills are sold with the F103C6** | Chip marking on the bench board | 32 KB flash fits the bench firmware but is tight for the full device (an estimated 22–25 KB). Specify the C8 (or CB) for the device and check the marking on every batch ([[STM32-Part-Selection]]) |
| **The device-role library is ≈8.5 KB on the Cortex-M3** | `lora-streetlight` at `1d97ae8`, arm-none-eabi-gcc 15.3, `-Os`: `lp_dev` 3.6 KB, message formats 2.0 KB, AES 1.4 KB, crypto 0.8 KB, codec, frame and airtime 0.7 KB. Tier-1 formats (1.7 KB) not needed on the device | With an SX1276 driver (~2–3 KB) it stays inside the 16 KB budget (O-03-6) |

Two practical notes for anyone repeating the bench: a USB-C Blue Pill needs a USB-A to USB-C cable to power up, and firmware started through the ROM bootloader's Go command must set `VTOR` itself, or its first interrupt jumps into the bootloader.

## Still to verify

- LED current against the dimming curve, with a 10 Ω 10 W dummy load or a 350 mA LED module.
- D4i: banks 202, 205 and 206 (bank 1 exists).
- `QUERY PHYSICAL MINIMUM` and the fade time index.
- System Failure Level on real bus loss: cut the bus supply for more than 550 ms (spec 05 O-05-2).
- Backward-frame delay ([[Open-Questions]] Q13): the RX timestamps can measure it directly.

## Related

- [[DALI-Fundamentals]]: the bus, timing and commands these results check
- [[DALI-Telemetry-And-D4i]]: capability discovery, run here on a real driver
- [[Fail-Safe-Lighting-Rule]]: layer 4 and the hole found here
- [[STM32-Part-Selection]]: the device platform and its budget
- [[Open-Questions]]: Q7, Q13, Q34, Q35

## Sources

- Device developer's bench session, 2026-10-28: serial logs and measurements. Repo `lrimando-dev/lora-streetlight-node`, branch `bench/dali-demo`: `docs/DALI2-DEMO-GUIDE.md`, `docs/DALI2-POC.md`, `bench/dali-demo/README.md`.
- LTECH SE-12-350-700-W1DS: DALI Alliance product listing 7484. Waveshare Pico-DALI2 schematic: https://files.waveshare.com/wiki/Pico-DALI2/Pico-DALI2.pdf

## History

- 2026-10-28: first bench results: rig, wiring, measurements and findings.

Back to [[Index]]
