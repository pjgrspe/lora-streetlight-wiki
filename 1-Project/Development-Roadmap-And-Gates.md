# Development Roadmap And Gates

The phased plan for the infrastructure side — server, master and slave firmware — with an **objective, testable exit gate** for each phase. Phases 0–3 are bench work; phases 4–5 are field work and sit outside the current Statement of Work.

## Key points

- **Specify before coding.** The register map, both protocol specs and the fail-safe table are signed before firmware starts.
- **Do the risky radio work early.** Real radios in a real box (coexistence, slot timing) are where estimates fail; prove them in the first weeks.
- **Bench before field.** Nothing goes to a pole until the destructive test matrix in Gate 3 is green.
- **Build the protocol simulator early and keep it as regression CI** — field debugging costs ~100x bench debugging.
- Version the wire protocol and the register map from day one.

## Phase 0 — Specification (no code)

Deliverables: wire-format spec for both tiers (frame header, addressing, opcodes, MIC, sequence numbers); Modbus register map as a generated CSV; channel plan and airtime budget; device and telemetry data model; fail-safe state machine; failure-mode table with the required behaviour per row; gateway hardware requirements.

**Gate 0:** the client signs the register map and fail-safe table; the device developer signs the tier-2 protocol; the hardware developer signs the hardware requirements. Write down "what do the lights do if X dies" for **every** X before writing firmware — see [[Failure-Modes-And-Mitigations]].

## Phase 1 — Bench: the cache and the wire

Modbus RTU slave serving a **static, fake** register image in under 10 ms; job state machine end-to-end against a simulated radio; registry in flash; console; timekeeping. The server's Modbus poller is built against the same fake image.

**Gate 1:** a Modbus master polls every register at 500 ms for 24 h with **zero timeouts and zero exceptions** while jobs are submitted and retired. A script verifies the register map matches the document.

*If you can't hit this with fake data, you'll never hit it with a radio.*

## Phase 2 — Bench: radios, cabled

Real SX1276 radios: master + one slave + device stubs, **RF-cabled with attenuators** — never over the air on the bench. Beacon/slot scheduling, time sync, scheduled device uplinks, retries, sequence numbers and MIC, duty-cycle ledger, alarm pre-emption. **Measure in-box antenna isolation** and prove the per-box TX scheduler; demonstrate primary→backup radio failover. Simulate a full 60-device cell.

**Gate 2:** latencies match the airtime model within ±30%; 10,000 injected commands produce **zero incorrect end states** under 20% packet loss with forced duplicates; the duty-cycle ledger never exceeds the limit; `PARTIAL` is reported correctly; failover works on both tiers.

## Phase 3 — Bench: full topology and autonomy

Master + multiple slaves + the device developer's real devices + simulated load up to the full device count. Autonomous fallback, store-and-forward with timestamped replay, alarm aggregation and rate-limiting, fault injection.

**Gate 3 — the important one.** A destructive test matrix, executed and logged: pull the RS485; kill the master; kill a slave; kill a radio; short a DALI link; drop a phase; set the RTC two hours wrong; jam a channel with a CW source; corrupt the registry flash; overflow the buffer.

For every case: lights behave per [[Fail-Safe-Lighting-Rule]], an alarm is raised, and the system self-recovers when the fault is removed — no manual intervention, no invented data. Plus a 7-day soak: no reboot, no leak, no counter overflow.

## Phase 4 — Field pilot: one section *(follow-on)*

Pick the **hardest** section (longest link, most foliage). One master, one slave, real meter and devices. Site survey record, the commissioning workflow exercised for real **including a deliberate device swap**, 24 h RF logging.

**Gate 4:** 30+ days at ≥99% telemetry delivery; zero unexplained dark-at-night incidents; every alarm reconciled against reality (false positives matter as much as false negatives); a technician who didn't build it diagnoses a seeded fault from the LEDs and CLI; a link-margin measurement taken in rain.

## Phase 5 — Full site and hardening *(follow-on)*

All sections. Full-scale load, per-luminaire timing verified, firmware update rehearsed one section at a time, spares provisioned, runbook written.

**Gate 5:** telemetry cadence meets spec at full scale; group ON is perceptually synchronised — **measure it with a video and a stopwatch**; 90-day review with the alarm log reconciled; a documented rollback for every update path; the hot-spare master proven by actually swapping it in.

## Phase 6 — Productisation *(ongoing)*

MQTT/Sparkplug or TALQ bridge on the server, multi-site, remote access, security review, certification if required.

## Cross-cutting rules

- **Version the wire protocol and register map** with explicit version fields: an older node with a newer gateway must interoperate or refuse cleanly — never silently misbehave.
- **The simulator is the regression suite.** Build it in Phase 1, extend it every phase.

## Related

- [[Commissioning-And-Field-Swap]]
- [[Open-Questions]]
- [[Team-And-Ownership]]

## Sources

- Research synthesis, 2026-09-09.

## History

- 2026-09-09 — roadmap written for a master-only engagement.
- 2026-10-14 — rescoped to server + master + slave firmware: server poller in Phase 1, coexistence and failover in Phase 2, multiple slaves and real devices in Phase 3; Phases 4–5 marked as follow-on.

Back to [[Index]]
