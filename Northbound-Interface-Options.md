# Northbound Interface Options

> **Updated 2026-10-14.** The client has confirmed **Modbus** on the server link, and the Local Monitoring Server is now **pgorospe's deliverable**. The suggested stack at the bottom of this note is refined in [[Local-Monitoring-Server-Design]] (FastAPI + PostgreSQL/TimescaleDB + embedded Grafana, rather than Node-RED as the core).

The master gateway sits exactly on the boundary that lighting standards care about. Since the monitoring server's stack is still undecided, the goal is to **keep the data model portable while implementing only Modbus now.**

## The options

| Option | Fit here |
|---|---|
| **Modbus RTU** | **What the diagram specifies, and what to build.** Simple, universally supported by SCADA. But register maps are flat and untyped, there are no events or alarms (only polling), no discovery, and 130 devices × 15 attributes is ~2,000 registers of hand-maintained map. **Workable for a fixed site; painful to evolve.** See [[Modbus-Register-Map-Design]]. |
| **Modbus TCP** | Strictly better than RTU *if the server can speak it* — same PDU, no 3.5T timing to fight, and transaction IDs let your 17-transaction sweep pipeline concurrently. **But the STM32WL has no Ethernet MAC**, so it needs a W5500-class SPI MAC (+$4 and an SPI bus) or a different MCU. |
| **MQTT + JSON** | **Best fit for telemetry.** Topics mirror the hierarchy (`site/{id}/section/{A}/{meter\|driver}/telemetry`), retained messages give last-known-value for free, QoS 1 gives at-least-once, LWT gives gateway-offline detection. Event-driven, so you stop polling. ⚠ **Needs TLS — mbedTLS is ~60–90 KB flash and 30–50 KB RAM, which does not fit the STM32WL.** |
| **REST/HTTP** | Good for configuration, commands and history queries; poor for high-rate telemetry. |
| **OPC UA** | **No.** Research is explicit that Cortex-M4/M7 parts lack the resources for a performant OPC UA server. The pragmatic pattern in the literature is exactly ours: **STM32 speaks Modbus; a more powerful box does OPC UA/MQTT translation.** |

**Recommendation: Modbus RTU on the gateway, and put the MQTT bridge on the Local Monitoring Server.** You get cloud reach and event semantics without a TLS stack in a 256 KB MCU. Design the data model for MQTT now — it costs nothing to keep the option open.

## TALQ — steal the model, don't implement it

TALQ defines the application-layer interface between a **Central Management System (CMS)** and the **gateway at the edge of an Outdoor Device Network**. Critically: *"TALQ does not constrain the protocols and network implementation within the ODN itself."*

**So TALQ says nothing about your LoRa link, your RS485 link, or your DALI buses. It standardises exactly the boundary the master gateway sits on.**

Technology: REST over HTTPS with JSON, defined as two OpenAPI v3 files plus a JSON Schema; the detailed spec has been freely available on GitHub since 2021 (certification is members-only).

**A TALQ device is a composite of Functions**, each with configuration / operational / metering / status attributes. The ones that map onto this system:

| TALQ Function | Maps to |
|---|---|
| **Basic** | Device ID, **longitude/latitude**, time zone, local time |
| **LampActuator** | Switching/dimming — `targetLightCommand` |
| **LampMonitor** | Energy, current, voltage, power, PF, temperature; lamp-failure and low/high-current events |
| **ElectricalMeter** | V/I/P/energy/PF, single or three phase ← **the 3 energy meters** |
| **CabinetConfiguration / CabinetMonitor** | ← **the slave gateway per section** |
| **SegmentMonitor** | Per-segment parameters ← **the per-phase sections** |
| **CommunicationController** | Comms attributes incl. **signal strength**, comms-failure and weak-signal events ← RSSI/SNR |
| **DriverAsset / LuminaireAsset** | ← **DALI Part 251 memory bank 1, almost 1:1** |

Services worth copying: on-demand read, **control via schedule-based "control programs" keyed on time and sensor inputs**, group management (groups may contain groups), and **data collection where the CMS configures what the ODN reports and when**.

**Verdict: do NOT implement TALQ on the STM32.** Instead: (a) borrow the object model for the registry and register map; (b) if TALQ is ever required by a municipal tender — and "more and more tenders require TALQ compatibility" — implement it as a **bridge on the Local Monitoring Server**, translating from Modbus/MQTT.

> **Name your internal objects after TALQ functions now. It is free, and it turns a possible future rewrite into a thin adapter.**

## The real corner-painting risk

It is not the protocol — it is the **data model**. Adopt a TALQ-shaped, MQTT-topic-friendly hierarchical object model in the registry and **generate the Modbus map from it**. Then any future northbound protocol is a translation layer on a real computer, not a firmware rewrite. See [[Device-Identity-Namespaces]].

## Suggested monitoring server stack (~130 devices, single site)

The server side is not this project's deliverable, but the gateway's design assumes something about it, so: the **Node-RED + Mosquitto + TimescaleDB + Grafana** stack on Docker Compose is the pragmatic default.

- One industrial mini-PC or a **Raspberry Pi 5 with NVMe** (avoid microSD — it dies of write wear), on a UPS, with a **cold imaged spare** since the site is typhoon-exposed.
- Node-RED's `node-red-contrib-modbus` makes the RS485 poller drag-and-drop; it normalises to TALQ-shaped JSON, publishes to Mosquitto, one flow writes the database and another evaluates rules.
- **TimescaleDB over InfluxDB** here, because the asset/relational side is substantial (pole ↔ driver ↔ GPS ↔ work orders) and one SQL database is simpler.
- **USB-RS485 adapter must be isolated**, with a genuine FTDI/CH340/CP210x chip. **Pin the device with a udev rule on serial number** (`/dev/ttyStreetlight`) — never rely on `/dev/ttyUSB0`.
- **Set the poller's response timeout ≥1 s.** A naive 100 ms Modbus timeout will fail constantly against a gateway that relays to LoRa — though the gateway should never actually block, per [[Cache-Behind-Synchronous-Wire-Pattern]].
- **Skip Ignition** unless a stakeholder mandates it; skip OPC UA; **skip cloud dependence** — schedules and the astronomical clock must live in the gateway so the site runs headless through a typhoon-induced WAN outage.

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Modbus-Register-Map-Design]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. TALQ white paper v2: https://www.talq-consortium.org/data/downloadables/5/2/4/2024-02-talq-white-paper-smart-city-protocol-specification-version-2.pdf · https://www.talq-consortium.org/2-why-talq/specification/ · OPC UA on constrained STM32: https://www.researchgate.net/publication/356950958 · STM32-as-Modbus-with-separate-translator pattern: https://www.mdpi.com/2227-7080/13/11/526
