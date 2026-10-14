# Local Monitoring Server Design

The "Central Monitoring **and Control**" PC. It polls the master gateway over Modbus RTU on RS485, stores history, shows operators the state of the site and lets them command it. **Since 2026-10-14 it is pgorospe's deliverable**, which means both ends of the Modbus contract are now in-house.

## What changes now that both ends are ours

- The register map is still worth writing down and generating from one source of truth ([[Modbus-Register-Map-Design]]). But the negotiation and sign-off overhead with a third-party server developer is gone.
- **Q6 is resolved.** The server will understand quality flags, `age_seconds`, and the asynchronous job pattern ([[Asynchronous-Command-Job-Pattern]]), so no defensive "compatibility mode" is needed.
- The FC03-only mirroring accommodation becomes optional. Keep it anyway; it's cheap and helps third-party tools like QModMaster during debugging.

## Responsibilities

| # | Function | Notes |
|---|---|---|
| S1 | **Modbus RTU poller** | Reads the gateway summary block every 1–2 s and does deep reads on `telemetry_epoch` change. **Response timeout ≥1 s.** Isolated USB-RS485 adapter pinned by serial number |
| S2 | **Historian** | Time-series for device status, power and energy. Event log. Command and job audit trail |
| S3 | **Dashboard** | Site → section → pole drill-down; online/stale/offline state per device; current level; alarms |
| S4 | **Control** | On/off/dim by pole, group, section or site; job progress showing `PARTIAL` honestly; maintenance overrides **with expiry** ([[Slave-Gateways-As-Intelligent-Sub-Controllers]]) |
| S5 | **Schedules** | Edit dusk/dawn offsets and dimming profiles, then push to the master. The gateways stay the authority at run time ([[Street-Lighting-Control-Logic]]) |
| S6 | **Alarms** | Raise, acknowledge, clear. Rate-limited and aggregated. Optional notification (email/SMS) |
| S7 | **Time source** | NTP-synced PC clock offered to the master as a *proposal* ([[Time-Distribution-And-Clock-Trust]]) |
| S8 | **Configuration as code** | Holds the intended configuration for every device and drives the master's "re-assert configuration" ([[Commissioning-And-Field-Swap]]) |
| S9 | **Users and reports** | A few roles (viewer / operator / admin). Energy report per section, exportable to CSV |

**Not in scope by default:** cloud/remote access, a mobile app, TALQ/MQTT northbound, multi-site. These are add-ons ([[Northbound-Interface-Options]]).

## Recommended stack

The earlier wiki suggestion was Node-RED + Mosquitto + TimescaleDB + Grafana. That's good for monitoring, but **operator control and job tracking are awkward to build well in Node-RED and Grafana.** Now that the server is a quoted deliverable, a small purpose-built app is easier to maintain and to hand over:

| Layer | Choice | Why |
|---|---|---|
| Poller + API | **Python (FastAPI) with `pymodbus`** | One process owns the serial port; the same library is used for the HIL Modbus tests ([[Development-Tooling]]) |
| Database | **PostgreSQL + TimescaleDB** | The relational side (pole ↔ device ↔ asset) is substantial; one database |
| UI | Lightweight web front-end served locally, with **Grafana embedded for the charts** | No need to hand-build charting |
| Packaging | Docker Compose on a Linux mini-PC, or native services on Windows if the client insists | ⚠ Confirm the client's PC and OS |

**Host hardware:** an industrial mini-PC with an SSD, on a UPS, with a **cold imaged spare** (the site is typhoon-exposed). Avoid microSD-based hosts.

## Design rule

**The server must never be in the nightly ON/OFF path.** If the PC is off for a week, the lights still follow the astronomical schedule in the master and slaves. The server is for supervision, override and history. See [[Fail-Safe-Lighting-Rule]] layer 1.

## Related

- [[Modbus-Register-Map-Design]]
- [[Northbound-Interface-Options]]
- [[Team-And-Ownership]]
- [[Index]]

## Sources

- Project owner, 2026-10-14: pgorospe now owns the Local Monitoring Server; server ↔ master link is Modbus. The diagram labels the server "Central Monitoring and Control".
