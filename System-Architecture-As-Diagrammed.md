# System Architecture (As Diagrammed)

The LoRa smart street lighting system as drawn in the architecture diagram. This note records the literal contents of the diagram and its handwritten annotations, kept separate from interpretation and design decisions.

> **Revised 2026-10-14.** The diagram file was re-issued with handwritten annotations from the client, and the project owner added verbal detail. Changes are marked **(rev 2026-10-14)**. The original reading is preserved under "History".

## Topology

```
Local Monitoring Server  ("Central Monitoring and Control")
   │  RS485 — Modbus                                              (rev 2026-10-14)
   ▼
MASTER LoRa GATEWAY ×1 — 2 LoRa radios ("1 is backup")            (rev 2026-10-14)
   │  LoRa tier 1 (×2 radios)
   ├──► SLAVE GW — Section 1 — 4 LoRa radios: 2 up, 2 down        (rev 2026-10-14)
   │        │ LoRa tier 2 (×2 radios, "for efficiency, ~50 devices")
   │        ├─ LoRa Energy Meter (integrated)
   │        └─ LoRa LED Driver + Monitoring Device(s) ──DALI──► 53 lights
   ├──► SLAVE GW — Section 2 ──► meter + LED driver device(s) ──DALI──► 38 lights
   └──► SLAVE GW — Section 3 ──► meter + LED driver device(s) ──DALI──► 37 lights
```

## Stated facts

| Item | Value |
|---|---|
| Sections / phases | 3 |
| Sub-gateways | 1 per phase (3 total) |
| **Server ↔ master protocol** | **Modbus over RS485** (rev 2026-10-14) |
| **Radios on the master** | **2**, master ↔ slaves. One is a backup; both may be active (rev 2026-10-14) |
| **Radios on each slave** | **4**: 2 to the master, 2 to the devices (rev 2026-10-14) |
| **Radio chip / MCU** | **SX1276** on **STM32** boards (rev 2026-10-14) |
| Integrated LoRa energy meters | 1 per phase (3 total) |
| Integrated LoRa LED driver + monitoring devices | Drawn as 1 per phase. **Annotated as "~50 devices" per slave**, i.e. per pole. ⚠ Interpretation, see below |
| DALI lights, phase 1 / 2 / 3 | 53 / 38 / 37 |
| Total DALI lights | 128 |

## Interpretation of the "~50 devices" annotation ⚠

The diagram body still shows **one** LED driver device per section, but the annotation next to the slave's downstream radios reads "2 for efficiency (50 devices)". The project owner relayed a concern about "maybe 50 devices connected at once" causing a bottleneck. The working interpretation is **one LoRa LED driver/monitoring device per pole** (≈128 device radios plus 3 meters). That is the per-pole branch of the old DALI topology fork. Consequences:

- **The DALI run length problem goes away**: each device drives a short DALI link inside its own pole ([[DALI-Bus-Length-Constraint]]).
- **The radio network grows from ~9 nodes to ~145**, which is the scenario the old SOW flagged as assumption A1 (+80–150 h). See [[Slave-Cell-Capacity-And-Scheduling]].
- Per-light fault *identity* comes from per-pole radio reports, not from DALI addressing across a long bus.

**Confirm with the client.** Tracked in [[Open-Questions]] Q3.

## Link types (per diagram legend)

- **RS485**: wired serial, server ↔ master. Now confirmed to carry **Modbus**.
- **LoRa radio**: two tiers. Master ↔ slaves, and slave ↔ devices. **Both tiers are now doubled** (two radios per link end).
- **DALI / lighting control**: LED driver device ↔ luminaire(s).

## Still not stated

- Physical distances between master, slaves and poles ([[Open-Questions]] Q16)
- Frequency band and channel plan; the regulatory envelope ([[Philippines-Regulatory-Context]])
- Which exact STM32 development board is supplied

Confirmed 2026-10-14: **one LED driver device per pole**, **one radio per device**, and a separate developer for the gateway hardware.

## Scope ownership (rev 2026-10-14)

pgorospe owns the **Local Monitoring Server, the Master Gateway, the three Slave Gateways, and the LoRa protocol**. One other developer owns the energy meter and the LED driver/monitoring device. See [[Team-And-Ownership]].

## History: reading of 2026-09-09 (superseded)

The original diagram named no protocol on RS485 and showed one radio per gateway. It showed exactly 2 endpoint devices per section (1 meter + 1 LED driver device driving a single DALI line), for **9 radio nodes in total**. Ownership was master gateway only.

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Open-Questions]]
- [[Index]]

## Sources

- `Sources/system-architecture-diagram-2026-09-09.jpg`. Original issued 2026-09-09; **re-issued with client annotations 2026-10-14** (same filename): "modbus" on the RS485 link, "and control" under Central Monitoring, "1 is backup / 2 LoRa radio" at the master, "4 LoRa" and "2 for efficiency (50 devices)" at slave 1.
- Verbal scope from project owner, 2026-09-09: STM32 for the board, custom (non-LoRaWAN) LoRa protocol.
- Verbal scope from project owner, 2026-10-14: server/master/slaves now owned by pgorospe; Modbus; dual radios; SX1276 + STM32; device side owned by one other developer.
