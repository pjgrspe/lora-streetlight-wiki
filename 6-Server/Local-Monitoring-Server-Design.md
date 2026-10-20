# Local Monitoring Server Design

The Local Monitoring Server ("Central Monitoring and Control") is the PC that polls the Master Gateway over Modbus RTU on RS485, stores history, shows operators the state of the site and lets them command it. It is pgorospe's deliverable, so both ends of the Modbus contract are built in-house. Its data model, alarm catalogue, dashboard and control requirements are specified in `Specs/specs/07-server-data-model.md` (v0.1).

## Key points

- **Modbus RTU over RS485** to the Master Gateway, 19200 8E1 by default. Response timeout ≥ 1 s; link down after 5 failed polls.
- **Stack:** Python 3.12 (FastAPI) + `pymodbus`, PostgreSQL 16 + TimescaleDB, and a web UI with **embedded Grafana** for charts.
- The server understands the gateway's **ages, comm states, event window and job pattern**, so no defensive "compatibility mode" is needed.
- **Spec 07 §6 is the single alarm catalogue.** Severity uses one scale everywhere: 0 info · 1 warning · 2 alarm · 3 critical.
- **The server is never in the nightly on/off path.** If the PC is off for a week, the gateways and devices still follow their astronomical schedules.
- Single site, local network, browser UI. Cloud, mobile and multi-site are out of scope.

## What being in-house changes

- The register map is still written down, versioned and **generated from one source of truth** ([[Modbus-Register-Map-Design]]), but there is no third-party server developer to negotiate with.
- The FC03-only mirroring accommodation in the register map becomes optional. Keep it anyway: it is cheap and lets tools like QModMaster read everything during debugging.

## Responsibilities

| # | Function | Notes |
|---|---|---|
| S1 | **Modbus RTU poller** | Every 2 s: the system head (with `heartbeat` and `telemetry_epoch`) and the event header. When the epoch changes: section summaries, section and meter blocks, then the whole block of each changed device. Every 15 min: every device block, read whole, ≤ 7 per request (9 requests per 60-device section). Events are stored, then acknowledged with `event_ack_seq`. Refuses to poll on a register-map major-version mismatch. Isolated USB-RS485 adapter, pinned by serial number |
| S2 | **Historian** | Time-series of device status, power and energy; event log; command and job audit trail |
| S3 | **Dashboard** | Site → section → pole drill-down; online / stale / offline state per device; current level; alarms |
| S4 | **Control** | On/off/dim by pole, group, section or site; job progress that shows `PARTIAL` honestly; maintenance overrides **with expiry** ([[Slave-Gateways-As-Intelligent-Sub-Controllers]]) |
| S5 | **Schedules** | Edit dusk/dawn offsets and dimming profiles, then push them to the Master. The gateways remain the run-time authority ([[Street-Lighting-Control-Logic]]) |
| S6 | **Alarms** | Raise, acknowledge, clear; rate-limited and aggregated (one grouped row for 12 lamp faults in a section, not 12). Gateway and device alarms map from their event codes; the server also derives its own (master offline/frozen, data stale, day-burner, lights-out estimate from the meter, phase unbalance against the design baseline, commanded-vs-measured power divergence, partial jobs) |
| S7 | **Time source** | NTP-synced PC clock, offered to the Master as a *proposal* it sanity-checks ([[Time-Distribution-And-Clock-Trust]]) |
| S8 | **Configuration as code** | Holds the intended configuration for every device and drives the Master's "re-assert configuration" ([[Commissioning-And-Field-Swap]]) |
| S9 | **Users and reports** | Viewer / operator / admin roles (up to 10 users, argon2 passwords, every action in an audit log); per-section energy, burning hours, availability, alarm summary; CSV export. Savings shown in kWh only unless the client confirms a metered tariff |

Optional later: email/SMS notification, remote access, MQTT/TALQ bridges ([[Northbound-Interface-Options]]).

## Stack

| Layer | Choice | Why |
|---|---|---|
| Poller + API | **Python (FastAPI) with `pymodbus`** | One process owns the serial port; the same library drives the Modbus test suite ([[Development-Tooling]]) |
| Database | **PostgreSQL + TimescaleDB** | Substantial relational side (pole ↔ device ↔ asset) plus time-series, in one database |
| UI | Lightweight web front-end, **Grafana embedded for charts** | No hand-built charting |
| Packaging | Docker Compose on a Linux mini-PC, or native services on Windows | ⚠ Client's PC and OS still to confirm |

Node-RED + Mosquitto + TimescaleDB + Grafana was considered. It is good for monitoring but awkward for operator control and job tracking, so it is not the core.

**Host hardware:** industrial mini-PC with an SSD, on a UPS, with a **cold imaged spare** (typhoon-exposed site). Avoid microSD-based hosts; they die of write wear.

**Data and retention (spec 07 §4–5):** device data stays attached to the **pole and address**, so history survives swaps (`device_assignment` keeps UID ↔ address ↔ pole history). Device telemetry is kept at full resolution for 90 days, then 15-minute aggregates for 2 years; meter data for 2 years; events, alarms, jobs and the audit log indefinitely. Nightly backups are kept 14 days, and a restore must be back in service within 2 hours.

**Serial link practice:** genuine FTDI/CP210x/CH340 isolated adapter, pinned with a udev rule (e.g. `/dev/ttyStreetlight`), never `/dev/ttyUSB0`.

## Related

- [[Modbus-Register-Map-Design]]
- [[Northbound-Interface-Options]]
- [[Team-And-Ownership]]

## Sources

- `Specs/specs/07-server-data-model.md`, `Specs/specs/01-modbus-register-map.md` §10 (v0.1)
- Project owner, 2026-10-14: pgorospe now owns the Local Monitoring Server; server ↔ master link is Modbus. The diagram labels the server "Central Monitoring and Control".

## History

- 2026-09-09: server assumed to be a third party's; Node-RED stack suggested in [[Northbound-Interface-Options]].
- 2026-10-14: server became pgorospe's deliverable; purpose-built FastAPI stack chosen; open question on server behaviour (old Q6) resolved.
- 2026-10-21: aligned with spec pack v0.1 (poller method from spec 01, alarm catalogue and severity scale from spec 07, retention, roles, backup targets).

Back to [[Index]]
