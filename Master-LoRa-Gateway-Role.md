# Master LoRa Gateway Role

> **Updated 2026-10-14.** The master now has **two LoRa radios** toward the slaves: one primary, one exercised standby, with P0 commands duplicated on both. See [[Dual-Radio-Redundancy-And-Coexistence]]. R2 now includes coordinating its own two radios (in-box TX blanking) and aligning tier-1 slots with each slave's cell superframe. The registry (R3) and cache (R4) now cover **up to ~140 devices (count TBC)** (per-pole LED devices + 3 meters), not 6. The Local Monitoring Server is now also pgorospe's deliverable ([[Local-Monitoring-Server-Design]]), so the "not the master's job" list below is now simply "the server's job". It's still in scope for pgorospe, just not in the gateway.

The master gateway is not "a radio bridge" — with no LoRaWAN network server in the design, it absorbs every function LoRaWAN would normally split across gateway + network server + join server + application server. It is a **miniature network server plus SCADA edge concentrator plus protocol translator in one STM32-class box**.

Underestimating this is the classic scoping error on private-LoRa projects.

## Responsibilities

| # | Responsibility | Priority |
|---|---|---|
| R1 | Southbound protocol translation: RS485/Modbus RTU slave ↔ custom LoRa frames. Owns the register map, scaling, word order. | P1 |
| R2 | Network coordination / MAC scheduling — the only authoritative transmit schedule. See [[LoRa-Airtime-Budget]]. | P1 |
| R3 | Device registry: UID, logical address, type, parent, section, keys, firmware version, commissioning state. See [[Device-Identity-Namespaces]]. | P1 |
| R4 | Telemetry aggregation + cache — this *is* the Modbus register image. See [[Cache-Behind-Synchronous-Wire-Pattern]]. | P1 |
| R5 | Command dispatch + confirmation: job queue, retries, per-target ACK, partial-success reporting. See [[Asynchronous-Command-Job-Pattern]]. | P1 |
| R6 | Time distribution — master is time root; drives all astronomical schedules. See [[Time-Distribution-And-Clock-Trust]]. | P1 |
| R7 | Network health monitoring: per-link RSSI/SNR, last-seen, retry/failure counters. See [[Gateway-Observability]]. | P1 |
| R8 | Alarm generation the endpoints cannot derive: node-missing, section-dark, phase imbalance, clock-unsynced. | P1 |
| R9 | Buffering during server outage: timestamped ring buffer, replayed on reconnect. | P2 |
| R10 | Configuration distribution, version-stamped so you can verify what each node actually holds. | P2 |
| R11 | Firmware distribution to slaves and endpoints. See [[Firmware-Update-Over-LoRa]]. | P3 |
| R12 | Security root: key store, frame counters, replay windows, join authorization. See [[Custom-Protocol-Security]]. | P1 minimal / P3 full |

## Explicitly NOT the master's job

Resist scope creep — these belong to the Local Monitoring Server:

- Long-term historian / time-series storage
- Operator authentication and UI
- Report generation
- Per-luminaire dimming curve math
- Any northbound standard protocol (TALQ, OPC UA) — see [[Northbound-Interface-Options]]

The design target: **dumb enough to be reliable, smart enough to run the site alone for a week.**

## Related

- [[System-Architecture-As-Diagrammed]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Fail-Safe-Lighting-Rule]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. TALQ v2 object model used as a responsibility checklist: https://www.talq-consortium.org/2-why-talq/specification/
