# Fail-Safe Lighting Rule

The single sentence that should govern every failure decision in this system:

> **Any failure whose effect on lighting is ambiguous resolves to LIGHTS ON at 100% during the local astronomical night.**

A dark road is a liability. Wasted energy is an invoice. The asymmetry is not close.

## Five independent layers

Each layer is independent of the one above it, so a failure at any level still lights the road.

| Layer | Failure | Fallback |
|---|---|---|
| 1 | Local Monitoring Server dies | Master runs the schedules |
| 2 | Master gateway dies | Slaves run local astro schedules — see [[Slave-Gateways-As-Intelligent-Sub-Controllers]] |
| 3 | Slave gateway dies | Driver device runs its own cached astro fallback over DALI |
| 4 | Driver device dies, or the DALI bus fails | **DALI driver's `System Failure Level` = 100%** — the driver acts on its own |
| 5 | Mains lost on a section | Nothing to do electrically; the value is fast, accurate alarming |

Layer 4 leans on DALI itself: in DALI-2, an interruption of the control interface longer than roughly 550 ms is defined as a **System Failure**, after which the driver outputs its stored 1-byte System Failure Level. Set it to 100%. (Timing is from secondary sources — verify against IEC 62386 text; see [[Open-Questions]].)

✅ **This layer survives the decision to use existing off-the-shelf drivers (2026-09-13).** `SYSTEM FAILURE LEVEL` and `POWER ON LEVEL` are mandatory in Part 102 across every DALI generation, so they work even on old DALI-1 gear. They are set by *command*, not by procurement — so this remains fully under your control. ⚠ But they are **per-driver configuration that does not survive a driver swap**, which makes the re-assert-configuration command in [[Commissioning-And-Field-Swap]] a safety feature, not just a convenience.

Also set **`Power-On Level` = 100%** so a mains restore at 2 a.m. lights the street immediately instead of waiting for the next schedule evaluation. The documented trap — "a non-last-level power-on setting means any power interrupt could light up an entire building" — is a *feature* for street lighting. Choose it deliberately and record why.

## Use the two levels as a diagnostic

Pick **distinct** values, e.g. `Power-On = 100%` and `System Failure = 90%`. A field technician can then tell which fault occurred by looking at the road.

## Corollaries elsewhere in the design

- **Never make ON/OFF a toggle** — see [[Asynchronous-Command-Job-Pattern]].
- **Commanded-off and no-power must never share a status code.** Conflating them makes "section dark" undetectable.
- **`CLOCK_UNTRUSTED` means lights on**, not lights off — see [[Time-Distribution-And-Clock-Trust]].
- The nightly on/off must cost **zero radio traffic** in the normal case, so a congested or dead network cannot prevent it.

## Related

- [[Failure-Modes-And-Mitigations]]
- [[DALI-Fundamentals]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. DALI-2 System Failure Level / Power-On Level behaviour: https://icnavigator.com/applications/lighting-led-drivers/dali-2-d4i-interface/ (secondary source — pending IEC 62386 verification)
