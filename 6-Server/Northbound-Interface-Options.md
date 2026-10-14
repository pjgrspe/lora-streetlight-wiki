# Northbound Interface Options

How the Master Gateway talks upward, and how the system could later talk to outside platforms. The decision is **Modbus RTU between gateway and server now**, with any MQTT or TALQ interface built as a **bridge on the server**, never in the gateway.

## Key points

- **Gateway ↔ server: Modbus RTU over RS485**, confirmed by the client. See [[Modbus-Register-Map-Design]].
- **MQTT, TALQ, OPC UA: on the server, if ever needed.** The gateway MCU has no Ethernet, and a TLS stack does not belong in it.
- **The real lock-in risk is the data model, not the protocol.** Name internal objects after TALQ functions and generate the Modbus map from that model.
- The server itself is pgorospe's deliverable: [[Local-Monitoring-Server-Design]].

## The options

| Option | Fit here |
|---|---|
| **Modbus RTU** | **Chosen.** Simple and universally supported. Weaknesses: flat untyped registers, no events (polling only), no discovery, a ~2,000-register map to maintain. Workable for a fixed site |
| **Modbus TCP** | Same PDU without the 3.5-character timing, and transactions can be pipelined. But the gateway MCU has no Ethernet MAC, so it would need a W5500-class SPI chip. Not needed |
| **MQTT + JSON** | Best fit for telemetry: topics mirror the hierarchy (`site/{id}/section/{n}/{meter\|driver}/telemetry`), retained messages give last-known value, LWT gives offline detection. Needs TLS (mbedTLS ≈ 60–90 KB flash, 30–50 KB RAM), so it belongs on the server |
| **REST/HTTP** | Good for configuration, commands and history; poor for high-rate telemetry |
| **OPC UA** | **No** on the MCU. Literature pattern is exactly this design: the STM32 speaks Modbus, a bigger box translates |

## TALQ: borrow the model, don't implement it

TALQ standardises the interface between a **Central Management System** and the **gateway at the edge of an outdoor device network**. It says nothing about the LoRa, RS485 or DALI links inside that network. It is REST over HTTPS with JSON (two OpenAPI v3 files plus a JSON Schema), freely available on GitHub since 2021; certification is members-only.

A TALQ device is a composite of Functions. The ones that map onto this system:

| TALQ Function | Maps to |
|---|---|
| **Basic** | Device ID, latitude/longitude, time zone, local time |
| **LampActuator** | Switching/dimming (`targetLightCommand`) ← the per-pole LED driver device |
| **LampMonitor** | Energy, current, voltage, power, PF, temperature; lamp-failure events |
| **ElectricalMeter** | V/I/P/energy/PF, single or three phase ← the section energy meter |
| **CabinetConfiguration / CabinetMonitor** | ← the Slave Gateway per section |
| **SegmentMonitor** | Per-segment parameters ← each section |
| **CommunicationController** | Signal strength, comms-failure and weak-signal events ← RSSI/SNR |
| **DriverAsset / LuminaireAsset** | ← DALI Part 251 memory bank 1, almost 1:1 |

Services worth copying: on-demand read, schedule-based "control programs", nested group management, and data collection where the CMS configures what is reported and when.

**Verdict:** do not implement TALQ on the gateway. Name internal objects after TALQ functions now (free), and if a tender ever requires TALQ, build it as a **server-side bridge** from Modbus/MQTT.

## Related

- [[Local-Monitoring-Server-Design]]
- [[Master-LoRa-Gateway-Role]]
- [[Modbus-Register-Map-Design]]

## Sources

- Research synthesis, 2026-09-09. TALQ white paper v2: https://www.talq-consortium.org/data/downloadables/5/2/4/2024-02-talq-white-paper-smart-city-protocol-specification-version-2.pdf · https://www.talq-consortium.org/2-why-talq/specification/ · OPC UA on constrained STM32: https://www.researchgate.net/publication/356950958 · STM32-as-Modbus-with-separate-translator pattern: https://www.mdpi.com/2227-7080/13/11/526

## History

- 2026-09-09: written when the server stack was undecided; recommended Modbus on the gateway and a Node-RED server.
- 2026-10-14: client confirmed Modbus; server became pgorospe's; server stack moved to [[Local-Monitoring-Server-Design]].

Back to [[Index]]
