# Local Monitoring Server Design

The Local Monitoring Server ("Central Monitoring and Control") is the PC that polls the Master Gateway over Modbus RTU on RS485, stores history, shows operators the state of the site and lets them command it. It is pgorospe's deliverable, so both ends of the Modbus contract are built in-house.

## Key points

- **Modbus RTU over RS485** to the Master Gateway. Poller response timeout **≥1 s**.
- **Stack:** Python (FastAPI) + `pymodbus`, PostgreSQL + TimescaleDB, and a web UI with **embedded Grafana** for charts.
- The server understands the gateway's **quality flags, `age_seconds` and asynchronous job pattern**, so no defensive "compatibility mode" is needed.
- **The server is never in the nightly on/off path.** If the PC is off for a week, the gateways and devices still follow their astronomical schedules.
- Single site, local network, browser UI. Cloud, mobile and multi-site are out of scope.

## What being in-house changes

- The register map is still written down, versioned and **generated from one source of truth** ([[Modbus-Register-Map-Design]]), but there is no third-party server developer to negotiate with.
- The FC03-only mirroring accommodation in the register map becomes optional. Keep it anyway: it is cheap and lets tools like QModMaster read everything during debugging.

## Responsibilities

| # | Function | Notes |
|---|---|---|
| S1 | **Modbus RTU poller** | Reads the gateway summary block every 1–2 s; deep-reads only when `telemetry_epoch` changes. Isolated USB-RS485 adapter, pinned by serial number |
| S2 | **Historian** | Time-series of device status, power and energy; event log; command and job audit trail |
| S3 | **Dashboard** | Site → section → pole drill-down; online / stale / offline state per device; current level; alarms |
| S4 | **Control** | On/off/dim by pole, group, section or site; job progress that shows `PARTIAL` honestly; maintenance overrides **with expiry** ([[Slave-Gateways-As-Intelligent-Sub-Controllers]]) |
| S5 | **Schedules** | Edit dusk/dawn offsets and dimming profiles, then push them to the Master. The gateways remain the run-time authority ([[Street-Lighting-Control-Logic]]) |
| S6 | **Alarms** | Raise, acknowledge, clear; rate-limited and aggregated |
| S7 | **Time source** | NTP-synced PC clock, offered to the Master as a *proposal* it sanity-checks ([[Time-Distribution-And-Clock-Trust]]) |
| S8 | **Configuration as code** | Holds the intended configuration for every device and drives the Master's "re-assert configuration" ([[Commissioning-And-Field-Swap]]) |
| S9 | **Users and reports** | Viewer / operator / admin roles; per-section energy report; CSV export |

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

**Serial link practice:** genuine FTDI/CP210x/CH340 isolated adapter, pinned with a udev rule (e.g. `/dev/ttyStreetlight`), never `/dev/ttyUSB0`.

## Related

- [[Modbus-Register-Map-Design]]
- [[Northbound-Interface-Options]]
- [[Team-And-Ownership]]

## Sources

- Project owner, 2026-10-14: pgorospe now owns the Local Monitoring Server; server ↔ master link is Modbus. The diagram labels the server "Central Monitoring and Control".

## History

- 2026-09-09: server assumed to be a third party's; Node-RED stack suggested in [[Northbound-Interface-Options]].
- 2026-10-14: server became pgorospe's deliverable; purpose-built FastAPI stack chosen; open question on server behaviour (old Q6) resolved.

Back to [[Index]]
