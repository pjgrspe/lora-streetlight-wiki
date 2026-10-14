# Development Tooling

The build, debug, test and logging setup for the gateway firmware, and the device-role protocol library shared with the device developer. The bottom line: CubeMX for init code, CMake + GCC for builds, Ceedling for host tests of everything protocol-related, and a hardware-in-the-loop bench with fake slaves and fake devices.

## Key points

- **CubeMX → CMake, `arm-none-eabi-gcc` + Ninja.** Commit and maintain the CMake yourself; no IDE lock-in.
- **Debug on the Nucleo's on-board ST-LINK**; a J-Link is worth it for RTT logging. The device developer's Blue Pills need a separate ST-LINK.
- **Ceedling + Unity + CMock on the host**, made possible by keeping `app/` free of HAL includes ([[Firmware-Architecture]]).
- **The same `app/` protocol code builds for both gateways and devices**: CI builds and tests the device role for the STM32F103 target too.
- **Build the protocol simulator early** and keep it as regression CI; field debugging costs ~100× bench debugging.

## Build

**CubeMX → CMake, built with `arm-none-eabi-gcc` + Ninja.** CubeMX handles pinmux, clocks and peripheral init for the STM32L4 (and for the F103 device target). Generate the CMake project once, then maintain it yourself: reproducible CI builds, `compile_commands.json` for clangd, host tests in the same tree. Keep **CubeIDE for debugging** (SWV/ITM, live expressions, fault analyser) attached to the ELF while editing in VS Code. Avoid **PlatformIO**: its STM32 support is Arduino-centric.

Flags: `-Og -g3` for development, `-Os -g3` for release. **Always keep `-g3` in release** and archive the ELF so field crash dumps can be symbolised. Plus `-ffunction-sections -fdata-sections -Wl,--gc-sections`, `-Wall -Wextra -Werror`, `-fstack-usage` (check worst-case stack per task), and `-Wl,-Map` with a **size report in CI** so flash growth shows up as a diff. This matters most for the **device-role library on the 64 KB Blue Pill**, which has a hard size budget ([[STM32-Part-Selection]]). Pin the toolchain version in CI and in a container.

## Debug probes

- **Nucleo on-board ST-LINK**: SWD + virtual COM port. Enough for most of the gateway work.
- **SEGGER J-Link** for one seat. **RTT** gives fast, non-intrusive `printf` over SWD (~1 µs per call), which is transformative when debugging Modbus framing or radio slot timing, where a blocking UART `printf` changes the behaviour you're trying to observe.
- **Blue Pill devices have no on-board debugger**: the device developer needs an ST-LINK/V2 or V3MINIE.
- On the final gateway board, bring out a **10-pin 1.27 mm Cortex debug header with SWO and NRST**, reachable with the enclosure closed if possible (gateway hardware requirements).

## Unit testing

**Ceedling + Unity + CMock.** The prerequisite matters more than the tool: **nothing in `app/` may include `stm32l4xx_hal.h`**. Then everything in `app/` runs on the host in CI.

- **100% branch coverage on the Modbus codec and register map**, including every malformed frame: short frame, bad CRC, wrong address, quantity 0, quantity 126, address wraparound, broadcast write.
- `lora_airtime.c` against the RP002 payload tables; `schedule.c` (NOAA sunrise/sunset) against published Manila tables.
- **Shared test vectors** (frame encodings, MIC, airtime, counter-window edges) used by both the gateway build and the device-role build, so the device developer runs the same known-answer tests.
- **The SX1276 continuous-RX FIFO-pointer handling** gets its own test against a fake radio ([[Continuous-Receive-Downlink]]).

**Renode** can run the real firmware binary in CI against a scripted UART peer, a useful middle tier between host tests and hardware. ⚠ Check Renode's STM32L4 platform support before relying on it.

## HIL fixture

Build it; you'll use it daily.

```
CI runner / dev PC
  ├── USB-RS485 adapter ──▶ Master RS485        (pymodbus: conformance + soak)
  ├── ST-LINK (on-board) ─▶ gateway SWD         (flash, logs, reset)
  ├── extra Nucleo + SX1276 ▶ over the air       "fake slave": respond / delay / corrupt / go silent / spam
  ├── extra Nucleo + SX1276 ▶ over the air       "fake devices": simulate a full 60-device cell
  ├── USB relay ──────────▶ gateway mains        (power-cycle; variac for BOR/PVD)
  └── RF attenuators + shielded box              link-margin sweeps, in-box isolation measurement
```

The radio sides are **cabled through attenuators** on the bench, never over the air, or you'll chase ghosts.

Suites, in priority order:

1. **Modbus conformance:** every FC, every exception, boundary quantities, malformed frames, and timing: a frame with a 2-character internal gap must be *rejected*; two frames separated by exactly t3.5 must both be *accepted*.
2. **Power-cut torture:** cut power at random points during config writes, NOR writes and an image install, **500+ times**, asserting the gateway always boots with valid config. This is the only way to know the ping-pong journal really works ([[Persistence-And-Storage]]).
3. **Radio degradation and failover:** fake slave goes silent, answers late, corrupts frames, answers twice; disconnect one radio of a pair. Assert the cache goes stale-flagged, counters move, **failover to the backup radio happens**, and **Modbus keeps answering throughout**.
4. **Cell load:** a simulated 60-device section reporting in its slots, including a mass re-join after a simulated mains restore ([[Slave-Cell-Capacity-And-Scheduling]]).
5. **Soak:** 72 h+, logging queue high-water marks, stack usage and error counters. **Monotonic growth is the leak detector.**

## Logging, three tiers

1. **Development: SEGGER RTT** over SWD (or SWO/ITM). Compile-time-filtered levels so release builds strip the strings; 300 log sites at ~40 B each is 12 KB you don't want to ship.
2. **Field, local: a RAM ring plus the serial console.** 4–8 KB, **binary-encoded** `{u32 tick, u8 level, u8 module, u16 code, u32 arg1, u32 arg2}` = 16 B per entry, with the human-readable strings held **off-device** in a table keyed by `code`. Dumpable via a console command *and* a Modbus mailbox opcode.
3. **Field, remote:** the Modbus diagnostic block plus a persisted fault log. The server polls `crash_count`, `iwdg_reset_count`, the per-radio counters and `fault_bitmap` every cycle and alarms on changes ([[Gateway-Observability]]).

**Never log from an ISR to anything that can block. Never log inside the Modbus response path. Rate-limit every log site that could fire in a loop.**

## Related

- [[Firmware-Architecture]]
- [[Gateway-Observability]]
- [[Development-Roadmap-And-Gates]]
- [[Team-And-Ownership]]

## Sources

- Research synthesis, 2026-09-09. Ceedling/Unity/CMock: https://github.com/throwtheswitch · https://embetronicx.com/tutorials/unit_testing/unit-testing-in-c-part-3-ceedling-installation/

## History

- 2026-09-09: written for an STM32WLE5 master gateway with one radio.
- 2026-10-14: retargeted to STM32L4 Nucleo gateways and Blue Pill devices. Added the device-role build target, shared test vectors, failover and 60-device cell tests, and in-box isolation measurement.

Back to [[Index]]
