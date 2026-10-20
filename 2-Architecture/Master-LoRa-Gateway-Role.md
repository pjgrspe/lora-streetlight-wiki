# Master LoRa Gateway Role

What the master gateway is responsible for. There is no LoRaWAN network server in this design, so the master is not a radio bridge: it is a **miniature network server, SCADA edge concentrator and protocol translator in one STM32 box**.

## Key points

- Sits between the Local Monitoring Server (Modbus RTU over RS485) and the Slave Gateways (LoRa tier 1), one Slave per section (configured 1–14; the actual count isn't fixed).
- Has **two SX1276 radios**: a primary and an exercised standby, with P0 commands sent on both. It runs one TX scheduler for both radios, because in-box radios can't transmit and receive at the same time ([[Dual-Radio-Redundancy-And-Coexistence]]).
- Owns the only authoritative tier-1 transmit schedule, the site-wide device registry and telemetry cache, command jobs, and time distribution.
- Design target: **dumb enough to be reliable, smart enough to run the site alone for a week.**
- Underestimating this scope is the classic scoping error on private-LoRa projects.

## Responsibilities

| # | Responsibility | Priority |
|---|---|---|
| R1 | Protocol translation: Modbus RTU slave ↔ custom LoRa frames. Owns the register map, scaling and word order. | P1 |
| R2 | Network coordination: the only authoritative tier-1 schedule. A beacon every 120 s cycle (heartbeat on the standby channel), then a 2 s slot per configured slave in GwID order. Each slave aligns its cell so its slot falls in its quiet window ([[Slave-Cell-Capacity-And-Scheduling]]). Coordinates its own two radios (in-box TX blanking) | P1 |
| R3 | Device registry for every device on the site (per-pole LED devices + meters, design ceiling 60 per Slave): UID, logical address, type, parent, section, keys, firmware version, commissioning state ([[Device-Identity-Namespaces]]). | P1 |
| R4 | Telemetry aggregation + cache, which *is* the Modbus register image ([[Cache-Behind-Synchronous-Wire-Pattern]]). | P1 |
| R5 | Command dispatch + confirmation: up to 8 open jobs, 10-min TTL, P0 group commands on both radios with `execute_at`, confirmation from slave CMD_ACKs, fail masks for `PARTIAL` ([[Asynchronous-Command-Job-Pattern]]). | P1 |
| R6 | Time distribution: the master is the time root for every astronomical schedule ([[Time-Distribution-And-Clock-Trust]]). | P1 |
| R7 | Network health: per-link and per-radio RSSI/SNR, last-seen, retry/failure counters, failover state ([[Gateway-Observability]]). | P1 |
| R8 | Events the endpoints can't derive (codes `0x8000`+): section lost, device offline, its own radio failover and standby loss, server lost, clock untrusted, config divergence, airtime reserve reached. Slave (`0x01`–`0x3F`) and device (`0x40`–`0x7F`) events are passed through unchanged. | P1 |
| R9 | Buffering during a server outage: a persisted event buffer (≥ 1000 events, 2000 recommended) with original timestamps, drained by the server through the Modbus event window and `event_ack_seq`. | P2 |
| R10 | Configuration distribution, version-stamped so you can verify what each node actually holds. | P2 |
| R11 | Firmware distribution to Slaves ([[Firmware-Update-Over-LoRa]]). Not in the current engagement. | P3 |
| R12 | Security root: holds the site key, derives per-slave and per-device keys, frame counters, replay windows, the device whitelist pushed to each slave ([[Custom-Protocol-Security]]). | P1 minimal / P3 full |

## Not the master's job

These belong to the Local Monitoring Server ([[Local-Monitoring-Server-Design]]), which pgorospe also delivers. They are still in scope, just not in the gateway:

- Long-term historian / time-series storage
- Operator authentication and UI
- Report generation
- Per-luminaire dimming-curve maths
- Any northbound standard protocol (TALQ, OPC UA) ([[Northbound-Interface-Options]])

## Related

- [[System-Architecture-As-Diagrammed]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Fail-Safe-Lighting-Rule]]

## Sources

- `Specs/specs/01-modbus-register-map.md`, `Specs/specs/02-tier1-protocol.md` (v0.1)
- Research synthesis, 2026-09-09. TALQ v2 object model used as a responsibility checklist: https://www.talq-consortium.org/2-why-talq/specification/

## History

- 2026-09-09: created for a master-only scope with one radio and ~6 endpoint devices.
- 2026-10-14: two radios on the master; registry covers per-pole devices; the server became pgorospe's deliverable.
- 2026-10-21: aligned with spec pack v0.1 (tier-1 cycle and slots, job limits, event code ranges, event buffer, key roles).

Back to [[Index]]
