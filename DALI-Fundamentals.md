# DALI Fundamentals

IEC 62386. What the master gateway needs to know: the addressing model, the command semantics its own command set must map onto, and the timing that bounds how fast telemetry can be gathered.

**There is no such thing as a DALI lamp.** A "DALI street light" is a luminaire whose constant-current LED driver has a DALI interface (a Device Type 6 driver per Part 207). The DALI pair lands on the driver's control terminals; mains lands separately.

## Bus physical layer

- **2-wire, polarity-free** (drivers rectify internally). Nominally **16 V DC**.
- Logic H: 11.5–20.5 V, receiver threshold >9.5 V. Logic L: −4.5 to +4.5 V, threshold <6.5 V. **6.5–9.5 V is an undefined band** in which a receiver must not decide.
- **250 mA max bus current**, enforced by the bus power supply's current limit. This makes the bus **inherently short-circuit tolerant** — shorting the pair is a legal signalling state, so a shorted bus just reads as logic 0 everywhere; it destroys nothing.
- **≤2 mA per control gear**, so the *current* budget permits 125 devices. **The 64 limit is an addressing limit (6-bit short address), not an electrical one.**
- Topology is free (bus, star, tree) but **no ring**.

## Data link timing — this bounds your polling

**Manchester encoded, 1200 bit/s.** Bit period 833.33 µs, half-bit `Te` = 416.67 µs.

| Frame | Composition | Duration |
|---|---|---|
| 16-bit forward (controller → gear) | 1 start + 2 bytes + 2 stop | **15.83 ms** |
| 24-bit forward (controller → control devices) | 1 start + 3 bytes + 2 stop | 22.5 ms |
| Backward (gear → controller) | 1 start + 1 byte + 2 stop | **9.17 ms** |

A backward frame is *optional*: "no answer" is a semantically distinct third state beyond the 256 byte values, and **collision** (multiple responders) is a fourth, deliberately exploited during addressing.

> **⚠ Sources conflict on the settling window — resolve before fixing your timeout constants.** Older/DALI-1 sources give forward→backward 2.92–9.17 ms and forward→forward ≥9.17 ms; DALI-2 training material aligned to Part 101 Ed.2/Ed.3 gives backward at 5.5–10.5 ms and forward→forward >13.5 ms. Could not be verified without the paid standard.
>
> **Engineering recommendation: wait the union.** Arm the receive window from 2.9 ms out to 11 ms, and space forward frames ≥13.5 ms.

**Consequence:** a query-response round trip is **~25 ms**. So:

- Polling 53 lights' status bytes serially = **~1.3 s per sweep, per bus**.
- `READ MEMORY LOCATION` returns **one byte** per ~25 ms round trip, so a 30-byte diagnostic bank is ~0.75 s and a **full diagnostic sweep of 53 lights is ~40 s**.

**This is why the master gateway must be a cache, not a pass-through** — see [[Cache-Behind-Synchronous-Wire-Pattern]]. Deep diagnostics must be scheduled, staggered and cached, never fetched on demand.

## Addressing

Address byte `YAAA AAAS`, data byte `XXXX XXXX`:

| Address byte | Meaning |
|---|---|
| `0AAA AAAS` | **Short address** 0–63 |
| `100A AAAS` | **Group address** 0–15 |
| `1111 111S` | **Broadcast** |
| `1010 0000`–`1111 1101` | Special commands (addressing, DTR loads) |

The **S bit** selects data-byte meaning: **S=0 → DAPC** (the byte *is* the arc power level); **S=1 → command opcode**. Control gear and control devices have **separate address universes**. Gear supports **16 groups and 16 scenes**, and may belong to multiple groups.

## Commands the gateway's command set maps onto

| Command | Notes |
|---|---|
| **DAPC** | The workhorse. **0 = OFF, 1 = physical min, 254 = 100%, 255 = MASK (no change)**. Fades using the stored fade time. |
| OFF | Immediate, no fade |
| **GO TO SCENE x** | Each gear recalls *its own* stored level for that scene. **The cheapest way to set 53 different levels with one broadcast frame — the single most valuable primitive for a bandwidth-starved LoRa link.** |
| RECALL MAX / MIN LEVEL | Jump to configured limits |
| STEP UP/DOWN, UP/DOWN | Relative — **avoid these over a lossy radio**; see [[Asynchronous-Command-Job-Pattern]] |

**Configuration parameters** (non-volatile in the driver, and **none of them survive a driver swap**):

- **MIN / MAX LEVEL** — clamp window. MIN cannot go below the read-only PHYSICAL MINIMUM. ⚠ Mixing drivers with different physical minima in one visual field gives **visibly uneven dimming at the bottom of the range** — a real roadway problem.
- **POWER ON LEVEL** and **SYSTEM FAILURE LEVEL** — the fail-safe hooks. Default 254 both. See [[Fail-Safe-Lighting-Rule]].
- **FADE TIME** (index 0–15, default 7 = 5.7 s) and **FADE RATE** (default 7 = 44.7 steps/s). Times follow √2 steps: index 10 = 16 s, 13 = 45.3 s, 15 = 90.5 s. DALI-2 adds extended fade time up to 16 min.

  **For street lighting, prefer long fades (16–90 s).** Abrupt level changes on a roadway read as flicker or failure to drivers, and a slow ramp also cuts inrush when 128 lights change level at once.

## The dimming curve — the thing most likely to bite you

- **Default is logarithmic**: 254 steps, ~2.8% per step, level 1 = 0.1%, level 254 = 100%. Approximates perceptual response, so **50% perceived is about level 180, not 127**.
- `level = 254·(1 + log10(P/100)/3)`, inverse `P = 100·10^(3·(level−254)/253)`.
- The curve defines **electrical arc power, not lumens and not perception.** Vendors and integrators conflate these constantly.
- DALI-2 / Part 207 permits an *optional* linear curve (0.394%–100% in equal power steps), inconsistently implemented and rarely useful.

> **Rule for the gateway data model: carry the level as the raw DALI 0–254 byte on the wire, and do the %↔level conversion in exactly one place, with curve type as a per-driver-model attribute.** Never let "50%" mean two things in two subsystems.

## Query commands and the status byte

`QUERY STATUS` (0x90) returns the minimum viable fault telemetry, and should map 1:1 into the gateway's per-light record:

| Bit | Meaning |
|---|---|
| 0 | **Control gear (driver) failure** |
| 1 | **Lamp failure** |
| 2 | Lamp arc power on |
| 3 | Limit error (commanded level was clamped) |
| 4 | Fade running |
| 5 | Reset state / not commissioned |
| 6 | **Missing short address** |
| 7 | Power cycle since last query |

Others: `QUERY ACTUAL LEVEL` (0xA0), `QUERY MAX/MIN/POWER ON/SYSTEM FAILURE LEVEL` (0xA1–A4), `QUERY PHYSICAL MINIMUM` (0x9A), `QUERY DEVICE TYPE` (0x99), `READ MEMORY LOCATION` (0xC5).

**Semantics to respect:** lamp and gear failure are **determined inside the driver** and merely reported — the controller cannot independently verify them. The bits clear on power cycle or after hysteresis, **so a "cleared" fault may be a power cycle, not a repair.** Poll counters, not just flags, if you want real fault history ([[DALI-Telemetry-And-D4i]]).

## Related

- [[DALI-Bus-Length-Constraint]]
- [[Street-Lighting-Control-Logic]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. DALI Alliance IEC 62386 parts: https://www.dali-alliance.org/standards/IEC62386.html · bus levels and opcodes: https://jared.geek.nz/2025/06/dali-lighting-protocol/ · Ozuno DALI Advanced Theory: https://ozuno.com/downloads/training/02%20-%20DALI%20ADVANCED%20THEORY.pdf · status byte: https://support.zencontrol.com/hc/en-us/articles/7093947995535
