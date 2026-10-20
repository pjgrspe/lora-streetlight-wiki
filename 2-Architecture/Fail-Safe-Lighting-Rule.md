# Fail-Safe Lighting Rule

The rule that governs every failure decision in this system: **any failure whose effect on lighting is ambiguous resolves to LIGHTS ON at 100% during the local astronomical night.** A dark road is a liability; wasted energy is an invoice. The full behaviour table, signed by the client and the device developer, is `Specs/specs/05-fail-safe-behaviour.md` (v0.1).

## Key points

- Five independent fallback layers, so a failure at any level still lights the road.
- Layer 4 is free: the DALI driver's own `SYSTEM FAILURE LEVEL` = 100%, which works on every DALI generation, including existing off-the-shelf drivers.
- Set `POWER ON LEVEL` = 100% too, so a mains restore at 2 a.m. lights the street immediately.
- Layer 3 (the per-pole device's own fallback) lives in **device-developer firmware**. Spec 05 lists it as obligations DD-1 to DD-11, each with a bench test, and the device developer signs it.
- By day, fallback layers keep the lights **off until dusk** (they still follow the astro schedule). Only an untrusted clock or a lost DALI link forces 100% at any hour.
- Corollaries: no ON/OFF toggles, commanded-off ≠ no-power, untrusted clock → lights ON, and the nightly switch costs zero radio traffic.

## Five independent layers

| Layer | Failure | Fallback |
|---|---|---|
| 1 | Local Monitoring Server dies | Master runs the schedules |
| 2 | Master Gateway dies | Slaves run local astro schedules ([[Slave-Gateways-As-Intelligent-Sub-Controllers]]) |
| 3 | Slave Gateway dies, or a device loses its Slave | The per-pole LED driver/monitoring device runs its own cached astro fallback over DALI |
| 4 | Device dies, or its DALI link fails | **DALI driver's `System Failure Level` = 100%**: the driver acts on its own |
| 5 | Mains lost on a section | Nothing to do electrically; the value is fast, accurate alarming |

Dual radios ([[Dual-Radio-Redundancy-And-Coexistence]]) reduce how often layers 2 and 3 are needed by covering radio, antenna and coax faults. They don't replace the layers, because a whole-box failure (PSU, lightning, MCU) still falls through to them.

## Layer 4 in detail

In DALI-2, a control-interface interruption longer than roughly **550 ms** is a **System Failure**, after which the driver outputs its stored 1-byte System Failure Level. Set it to 100%. ⚠ Timing is from secondary sources; verify against the IEC 62386 text ([[Open-Questions]]).

- `SYSTEM FAILURE LEVEL` and `POWER ON LEVEL` are mandatory in Part 102 across every DALI generation, so they work on old DALI-1 gear too. They are set by *command*, not by procurement.
- ⚠ They are **per-driver configuration that does not survive a driver swap**. That makes the re-assert-configuration command in [[Commissioning-And-Field-Swap]] a safety feature, not just a convenience.
- The documented trap, "a non-last-level power-on setting means any power interrupt lights up the whole building", is a *feature* for street lighting. Choose it deliberately and record why.

**Default: both levels = 254 (100%).** A diagnostic variant is offered to the client as a choice: `System Failure Level = 250` (≈90%), so a field technician can tell a device/DALI failure from a power-up by looking at the road. The difference is barely visible to drivers.

## Corollaries elsewhere in the design

- **Overrides always expire** (max 720 min) and are cleared when a slave goes `AUTONOMOUS` or any node reboots.
- **A late command is never executed:** radio commands carry `not_after`.

- **Never make ON/OFF a toggle** ([[Asynchronous-Command-Job-Pattern]]).
- **Commanded-off and no-power must never share a status code.** Conflating them makes "section dark" undetectable.
- **`CLOCK_UNTRUSTED` means lights ON**, not off ([[Time-Distribution-And-Clock-Trust]]).
- **The nightly on/off costs zero radio traffic** in the normal case, so a congested or dead network can't prevent it.

## Related

- [[Failure-Modes-And-Mitigations]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[DALI-Fundamentals]]

## Sources

- `Specs/specs/05-fail-safe-behaviour.md` (v0.1, 2026-10-20)
- Research synthesis, 2026-09-09. DALI-2 System Failure Level / Power-On Level behaviour: https://icnavigator.com/applications/lighting-led-drivers/dali-2-d4i-interface/ (secondary source — pending IEC 62386 verification)

## History

- 2026-09-13: confirmed that layer 4 survives the decision to use existing off-the-shelf DALI drivers.
- 2026-10-14: layer 3 now sits in the per-pole devices (device-developer firmware).
- 2026-10-21: aligned with spec pack v0.1 (both DALI levels 254 by default, 250 diagnostic as a client choice; day/night meaning of lights ON; device obligations in spec 05).

Back to [[Index]]
