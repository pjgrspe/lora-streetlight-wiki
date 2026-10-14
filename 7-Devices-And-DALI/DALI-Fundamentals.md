# DALI Fundamentals

DALI (IEC 62386) is the digital control interface between each per-pole LoRa LED driver device and its luminaire's LED driver. This note covers what the system's data model and command set must respect: addressing, command semantics, timing and the dimming curve.

## Key points

- A "DALI street light" is **a luminaire whose LED driver has a DALI interface** (Device Type 6, Part 207). There is no DALI lamp.
- In this system each DALI link is **short and inside one pole**, run by the per-pole device (device developer). The radio carries the distance. See [[DALI-Bus-Length-Constraint]].
- **DALI is slow: ~25 ms per query-response, one byte per memory read.** Deep diagnostics must be scheduled and cached by the device, never fetched on demand over LoRa.
- **Carry the level as the raw 0–254 byte everywhere**; convert to % in exactly one place. The default curve is logarithmic: 50% perceived ≈ level 180.
- `POWER ON LEVEL` and `SYSTEM FAILURE LEVEL` are the fail-safe hooks, mandatory in every DALI generation. See [[Fail-Safe-Lighting-Rule]].

## Bus physical layer

- **2-wire, polarity-free**, nominally **16 V DC**.
- Logic H 11.5–20.5 V (threshold >9.5 V); logic L −4.5 to +4.5 V (threshold <6.5 V). **6.5–9.5 V is undefined.**
- **250 mA max bus current**, limited by the bus power supply, so a shorted bus is a legal "logic 0", not damage.
- **≤2 mA per control gear.** The 64-device limit is an **addressing** limit (6-bit short address), not electrical.
- Free topology (bus, star, tree), **no ring**.

## Timing

**Manchester, 1200 bit/s.** Bit 833.33 µs, half-bit `Te` 416.67 µs.

| Frame | Composition | Duration |
|---|---|---|
| 16-bit forward (controller → gear) | 1 start + 2 bytes + 2 stop | **15.83 ms** |
| 24-bit forward (controller → control devices) | 1 start + 3 bytes + 2 stop | 22.5 ms |
| Backward (gear → controller) | 1 start + 1 byte + 2 stop | **9.17 ms** |

A backward frame is optional: "no answer" is a distinct state, and **collision** (several responders) is another, used during addressing.

⚠ **Settling windows conflict between sources.** DALI-1-era: forward→backward 2.92–9.17 ms, forward→forward ≥9.17 ms. DALI-2 (Part 101 Ed.2/3) material: backward 5.5–10.5 ms, forward→forward >13.5 ms. Unverifiable without the paid standard. **Wait the union:** receive window 2.9–11 ms, forward spacing ≥13.5 ms.

**Consequences:**

- Query-response round trip ≈ **25 ms**.
- `READ MEMORY LOCATION` returns **one byte per round trip**: a 30-byte diagnostic bank ≈ 0.75 s per driver.
- With one driver per pole this is cheap locally, but the result still has to cross LoRa, so the device reads, caches and reports on its own schedule. See [[Cache-Behind-Synchronous-Wire-Pattern]].

## Addressing

Address byte `YAAA AAAS`, data byte `XXXX XXXX`:

| Address byte | Meaning |
|---|---|
| `0AAA AAAS` | **Short address** 0–63 |
| `100A AAAS` | **Group address** 0–15 |
| `1111 111S` | **Broadcast** |
| `1010 0000`–`1111 1101` | Special commands (addressing, DTR loads) |

**S bit:** S=0 → **DAPC** (the data byte is the level); S=1 → command opcode. Control gear and control devices have **separate address universes**. Gear supports **16 groups and 16 scenes**.

On an in-pole link with one driver, broadcast or a fixed short address is enough. Site-wide groups and scenes are now implemented in the radio protocol (group commands with `execute_at`), not across a long DALI bus. See [[Popcorn-Effect-And-Scheduled-Execution]].

## Commands the system maps onto

| Command | Notes |
|---|---|
| **DAPC** | The workhorse. **0 = OFF, 1 = physical min, 254 = 100%, 255 = MASK (no change).** Fades using the stored fade time |
| OFF | Immediate, no fade |
| **GO TO SCENE x** | Each driver recalls its own stored level. Useful for per-pole presets set once at commissioning |
| RECALL MAX / MIN LEVEL | Jump to configured limits |
| STEP UP/DOWN, UP/DOWN | Relative. **Avoid over a lossy radio**: a retry doubles the step. See [[Asynchronous-Command-Job-Pattern]] |

**Configuration (non-volatile in the driver; none of it survives a driver swap):**

- **MIN / MAX LEVEL**: clamp window. MIN cannot go below the read-only PHYSICAL MINIMUM. ⚠ Mixed driver models with different physical minima in one view give **visibly uneven dimming at the low end**.
- **POWER ON LEVEL** and **SYSTEM FAILURE LEVEL**: fail-safe hooks, default 254.
- **FADE TIME** (index 0–15, default 7 = 5.7 s) and **FADE RATE** (default 7 = 44.7 steps/s). √2 steps: index 10 = 16 s, 13 = 45.3 s, 15 = 90.5 s. DALI-2 adds extended fade up to 16 min. **Prefer long fades (16–90 s) on roadways**: abrupt changes read as flicker, and slow ramps cut inrush when many lights change together.

## The dimming curve

- **Default is logarithmic**: 254 steps, ~2.8% per step, level 1 = 0.1%, level 254 = 100%.
- `level = 254·(1 + log10(P/100)/3)`; inverse `P = 100·10^(3·(level−254)/253)`.
- The curve defines **electrical arc power**, not lumens and not perception.
- DALI-2 / Part 207 allows an optional linear curve, inconsistently implemented.

**Data-model rule:** raw 0–254 byte on the wire, % conversion in exactly one place, curve type stored per driver model. "50%" must never mean two things.

## Status byte and queries

`QUERY STATUS` (0x90) is the minimum viable fault telemetry, mandatory in every DALI generation:

| Bit | Meaning |
|---|---|
| 0 | **Control gear (driver) failure** |
| 1 | **Lamp failure** |
| 2 | Lamp arc power on |
| 3 | Limit error (commanded level clamped) |
| 4 | Fade running |
| 5 | Reset state / not commissioned |
| 6 | **Missing short address** |
| 7 | Power cycle since last query |

Others: `QUERY ACTUAL LEVEL` (0xA0), `QUERY MAX/MIN/POWER ON/SYSTEM FAILURE LEVEL` (0xA1–A4), `QUERY PHYSICAL MINIMUM` (0x9A), `QUERY DEVICE TYPE` (0x99), `READ MEMORY LOCATION` (0xC5).

Lamp and gear failure are **decided inside the driver**; the controller cannot verify them. Bits clear on power cycle or after hysteresis, **so a cleared fault may be a power cycle, not a repair.** Poll counters for real history ([[DALI-Telemetry-And-D4i]]).

## Related

- [[DALI-Bus-Length-Constraint]]
- [[DALI-Telemetry-And-D4i]]
- [[Street-Lighting-Control-Logic]]
- [[Fail-Safe-Lighting-Rule]]

## Sources

- Research synthesis, 2026-09-09. DALI Alliance IEC 62386 parts: https://www.dali-alliance.org/standards/IEC62386.html · bus levels and opcodes: https://jared.geek.nz/2025/06/dali-lighting-protocol/ · Ozuno DALI Advanced Theory: https://ozuno.com/downloads/training/02%20-%20DALI%20ADVANCED%20THEORY.pdf · status byte: https://support.zencontrol.com/hc/en-us/articles/7093947995535

## History

- 2026-09-09: written for one DALI bus per section driven from a section controller.
- 2026-10-14: system went per-pole; DALI links are now in-pole and run by the device developer's device. Timing and group sections adapted.

Back to [[Index]]
