# Development Tooling

## Build

**Recommendation: CubeMX → CMake, built with `arm-none-eabi-gcc` + Ninja.** Use CubeMX for pinmux, clock tree and peripheral init — it is the only tool that gets the STM32WL clock tree and `SUBGHZSPI` right first try. Generate the CMake project once, then commit and maintain the CMake yourself. You get reproducible CI builds, `compile_commands.json` for clangd, host unit tests in the same tree, and no IDE lock-in.

Keep **CubeIDE for debugging** (SWV/ITM, live expressions, fault analyzer) attached to the ELF, while editing in VS Code. Avoid **PlatformIO** for a product you'll maintain for a decade — its STM32 support is Arduino-centric and lags ST.

Flags: `-Og -g3` for development, `-Os -g3` for release — **always keep `-g3` in release** and archive the ELF with each release so you can symbolise a field crash dump. Plus `-ffunction-sections -fdata-sections -Wl,--gc-sections`, `-Wall -Wextra -Werror`, `-fstack-usage` (then check worst-case per-task stack against your allocations), and `-Wl,-Map` with a **size report in CI** so flash overflow shows up as a diff, not as a link failure the week of release. Pin the toolchain version in CI and in a container.

## Debug probes

- **ST-LINK/V3MINIE** (~$10) — SWD + VCP + SWO. Enough for 95% of the work.
- **SEGGER J-Link** for one seat, worth the money for two reasons: **RTT** gives megabit-rate, non-intrusive `printf` over SWD with no UART and ~1 µs per call — transformative when debugging timing-sensitive Modbus framing, where a blocking UART `printf` changes the behaviour you're trying to observe; plus unlimited flash breakpoints and much faster flash download.

Bring out a **10-pin 1.27 mm Cortex Debug header with SWO connected** and `NRST` on it — and keep it reachable with the enclosure closed if you can. A gland-plugged access port pays for itself the first time a gateway misbehaves in the field.

## Unit testing

**Ceedling + Unity + CMock.** The architectural prerequisite matters more than the tool: **nothing in `app/` may include `stm32wlxx_hal.h`** — see the layering in [[Firmware-Architecture]]. Then everything in `app/` runs on the host in CI.

Target **100% branch coverage on the Modbus codec and register map**, including every malformed-frame case: short frame, bad CRC, wrong address, quantity = 0, quantity = 126, address wraparound at 0xFFFF, broadcast write. These are precisely the frames a real master will eventually send, and precisely the cases that hang a hand-rolled parser.

`schedule.c` (the NOAA sunrise/sunset code) is trivially testable against published tables for Manila. `lora_airtime.c` is testable against the RP002 payload tables.

**Renode** can run the actual firmware binary in CI against a scripted UART peer — a useful middle tier between host tests and hardware.

## HIL fixture

Build it; you will use it daily.

```
CI runner / dev PC
  ├── USB-RS485 adapter ──▶ gateway RS485    (pymodbus: conformance + soak)
  ├── ST-LINK/V3 ────────▶ gateway SWD       (flash, RTT logs, reset)
  ├── 2nd STM32WL board ─▶ over the air      "fake slave": respond / delay /
  │                                           corrupt / go silent / spam
  ├── USB relay ─────────▶ gateway mains     (power-cycle; variac for BOR/PVD)
  └── RF attenuator + shielded box            link-margin sweeps
```

Suites, in priority order:

1. **Modbus conformance** — every FC, every exception, every boundary quantity, malformed frames, and the timing cases: a frame with a 2-character internal gap must be *rejected*; two frames separated by exactly t3.5 must both be *accepted*.
2. **Power-cut torture** — kill mains at random points during a flash config write, a NOR OTA write, and a bootloader install, **500+ times**, asserting the device always boots to a working image with valid config. **This is the test that decides whether the product survives deployment**, and the only way to find out whether your ping-pong journal is actually correct ([[Persistence-And-Storage]]).
3. **Radio degradation** — fake slave goes silent, answers late, answers with a corrupt CRC, answers twice. Assert the cache goes stale-flagged, counters move, the escalation ladder fires, and **Modbus keeps answering throughout**.
4. **Soak** — 72 h+, logging queue high-water marks, stack/heap usage and error counters. **Monotonic growth is your leak detector.**
5. **OTA** — full cycle, plus interrupted transfers, a bad signature, a downgrade attempt, and a never-marks-healthy rollback.

**Build the protocol simulator in Phase 1 and keep it as regression CI.** Field debugging costs roughly 100x bench debugging — see [[Development-Roadmap-And-Gates]].

## Logging — three tiers

1. **Development: SEGGER RTT** over SWD (or SWO/ITM on ST-LINK). Compile-time-filtered levels so release builds compile the strings out — a format string is ~40 bytes of flash each, and 300 log sites is 12 KB you don't want to ship.
2. **Field, local: a RAM ring plus an LPUART console.** 4–8 KB circular, **binary-encoded** `{u32 tick, u8 level, u8 module, u16 code, u32 arg1, u32 arg2}` = 16 B/entry, ~500 entries, with the human-readable strings held **off-device** in a table keyed by `code`. 16 bytes per event instead of 60+, and a technician with a USB-serial cable and a decoder script gets full history. Dumpable via a console command *and* a Modbus mailbox opcode.
3. **Field, remote:** the Modbus diagnostic block plus a persisted fault log. The server should poll `crash_count`, `iwdg_reset_count`, the radio counters and `fault_bitmap` every cycle and alarm on deltas.

**Never log from an ISR to anything that can block. Never log inside the Modbus response path. Rate-limit every log site that could fire in a loop** — an error logging at 1 kHz will fill your NOR and take the wear budget with it.

## Related

- [[Firmware-Architecture]]
- [[Gateway-Observability]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Ceedling/Unity/CMock: https://github.com/throwtheswitch · https://embetronicx.com/tutorials/unit_testing/unit-testing-in-c-part-3-ceedling-installation/
