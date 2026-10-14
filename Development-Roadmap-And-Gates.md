# Development Roadmap And Gates

> **Updated 2026-10-14.** The phases below were written for a master-only engagement. Under the v5 whole-infrastructure scope (`Output/LoRa-System-Statement-of-Work.md`), Phase 1 also builds the server poller against the fake register image. Phase 2 adds slave firmware, the dual-radio coexistence measurement and a simulated 54-device cell. Phase 3 is the full bench topology (master + 3 slaves + the device developer's real devices + simulated load to the full device count (up to ~140)). Phases 4–5 (field pilot, full site) remain follow-on work.

Phased plan for the master gateway. Each phase has an **objective, testable exit gate**. Do not proceed on vibes.

## Phase 0 — Specification (1–2 weeks, no code)

Deliverables: wire-format spec (frame header, addressing, opcodes, MIC, sequence numbers); Modbus register map v0.1 as a generated CSV; device and telemetry data model; the fail-safe state machine; the airtime budget spreadsheet; the failure-mode table with required behaviour per row.

**Gate 0:** the customer *and* the server developer both sign the register map and the fail-safe behaviour table.

Write down the answer to "what do the lights do if X dies" for **all** X before writing firmware. Most of this project's risk is retired here, cheaply. See [[Failure-Modes-And-Mitigations]].

## Phase 1 — Bench: the cache and the wire (3–4 weeks)

Modbus RTU slave serving a **static, fake** register image in under 10 ms; job/ticket state machine end-to-end against a simulated radio; registry in flash; console CLI; timekeeping.

**Gate 1:** a Modbus master (Modpoll / QModMaster) polls every register at 500 ms for 24 h with **zero timeouts and zero exceptions**, while jobs are being submitted and retired. An automated script verifies the register map matches the document.

*If you cannot hit this with fake data, you will never hit it with a radio.*

## Phase 2 — Bench: two-hop radio, cabled (4–6 weeks)

Real LoRa: master + 1 slave + 1 endpoint stub, **RF-cabled with attenuators** — never over the air on the bench, or you will chase ghosts. Beacon/slot scheduling, time distribution, report-on-change plus heartbeat, retries, sequence numbers and MIC, ack-with-state, duty-cycle ledger, alarm priority pre-emption.

**Gate 2:** measured end-to-end latencies match the airtime model within ±30%; 10,000 injected commands produce **zero incorrect end states** under 20% artificial packet loss with forced duplicates; the duty-cycle ledger provably never exceeds the limit; `PARTIAL` is correctly reported.

*Instrument and log everything now — you will not get this visibility in the field.*

## Phase 3 — Bench: full topology, DALI, autonomy (4–6 weeks)

3 slaves, 6 endpoints, 8+ real DALI luminaires plus a simulated 128 for load. Autonomous fallback, store-and-forward with timestamped replay, alarm aggregation and rate-limiting, DALI fault injection, A/B firmware update of one slave.

**Gate 3 — the important one.** A **destructive test matrix**, executed and logged: pull the RS485; kill the master; kill a slave; short the DALI bus; drop a phase; set the RTC two hours wrong; jam the channel with a CW source; corrupt the registry flash; overflow the buffer.

For every case: lights behave per [[Fail-Safe-Lighting-Rule]], an alarm is raised, and the system self-recovers when the fault is removed — with no manual intervention and no invented data. Plus a 7-day soak: no reboot, no memory leak, no counter overflow.

*Nothing goes to a pole until this table is green.*

## Phase 4 — Field pilot: one section (4–8 weeks in the field)

Pick the **hardest** section (longest link, most foliage), not the easiest. One master, one slave, real meter, real driver, ~40 luminaires. Site survey record, the commissioning workflow exercised for real **including a deliberate device swap**, 24 h RF logging.

**Gate 4:** 30+ consecutive days at ≥99% telemetry delivery at the specified cadence; zero unexplained dark-at-night incidents; every alarm reconciled against reality — **no false positives matters as much as no false negatives**, because operators stop reading a noisy alarm list and then you effectively have no alarms; a technician who did not build it diagnoses a seeded fault using only the LEDs and the CLI; a link-margin measurement taken in rain.

## Phase 5 — Full site and hardening (4–8 weeks)

All 3 sections, 128 lights. Full-scale load, per-luminaire dump timing verified, FUOTA rehearsed one section at a time, spares provisioned, runbook written.

**Gate 5:** telemetry cadence meets spec at full scale; group ON is perceptually synchronised — **measure it with a video and a stopwatch**, because that is the customer's real acceptance test whether they say so or not; 90-day operational review with the alarm log reconciled; a documented rollback for every update path; the hot-spare master proven by actually swapping it in.

## Phase 6 — Productisation (ongoing)

Northbound MQTT/Sparkplug or a TALQ bridge on the server, multi-site, remote access, security review, certification if required.

## Cross-cutting, from day one

- **Version the wire protocol and the register map** with explicit version fields and a compatibility rule: an older slave with a newer master must interoperate, or refuse cleanly — never silently misbehave.
- **Build a protocol simulator/harness in Phase 1 and keep it as regression CI.** The highest-leverage investment on a project like this: field debugging costs ~100x bench debugging.

## Related

- [[Commissioning-And-Field-Swap]]
- [[Open-Questions]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09.
