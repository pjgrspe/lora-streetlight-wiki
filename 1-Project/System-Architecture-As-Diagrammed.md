# System Architecture (As Diagrammed)

What the client's architecture diagram and its handwritten annotations actually say, kept separate from design decisions. The system is a two-tier LoRa network: one Master Gateway talks to one Slave Gateway per section, and each Slave talks to the devices on its poles.

## Key points

- Server ↔ Master is **Modbus over RS485** — the only wired data link.
- The Master has **2 LoRa radios** ("1 is backup"); each Slave has **4** (2 up to the master, 2 down to devices).
- **One LoRa LED Driver & Monitoring Device per pole** (confirmed), plus one energy meter per section. Exact device count TBC; design ceiling 60 per Slave.
- **The number of sections is not fixed.** The diagram shows 3 (53 / 38 / 37 lights, 128 total), but there may be 4 or more.
- Radios are **SX1276**, boards are **STM32**.
- The design that answers this diagram is now written as the M1 spec pack (`Specs/specs/00`–`07`). Both radio tiers run on a **120 s cycle** by default (spec 04).

## Topology

```
Local Monitoring Server  ("Central Monitoring and Control")
   │  RS485 — Modbus
   ▼
MASTER LoRa GATEWAY ×1 — 2 LoRa radios (primary + backup)
   │  LoRa tier 1
   ├──► SLAVE GW — section A — 4 radios (2 up, 2 down)
   │        │  LoRa tier 2 (2 channels)
   │        ├─ LoRa Energy Meter ×1
   │        └─ LoRa LED Driver & Monitoring Device ×1 per pole ──DALI──► its luminaire
   ├──► SLAVE GW — section B ──► …
   └──► SLAVE GW — … (one per section; count TBC)
```

## What the diagram states

| Item | Value |
|---|---|
| Server ↔ master protocol | Modbus over RS485 |
| Radios on the master | 2 — one is a backup; both may be active |
| Radios on each slave | 4 — 2 to the master, 2 to the devices |
| Radio chip / MCU | SX1276 on STM32 |
| Sub-gateways | 1 per section |
| Energy meters | 1 per section |
| LED driver & monitoring devices | Drawn as 1 per section; annotated "~50 devices" per slave → one per pole |
| DALI lights in the diagram | 53 / 38 / 37 = 128 across 3 sections |

## Link types (diagram legend)

- **RS485** — wired serial, server ↔ master, carrying Modbus.
- **LoRa radio** — two tiers: master ↔ slaves and slave ↔ devices. Both tiers are doubled (two radios per link end on the gateways).
- **DALI / lighting control** — LED driver device ↔ its luminaire.

## Interpretation of the annotations

- **"~50 devices" → one device per pole.** The body of the diagram still shows one LED driver device per section, but the annotation and the client's bottleneck concern mean a device on every pole. Consequences:
  - The DALI run-length problem disappears — each device drives a short DALI link inside its own pole ([[DALI-Bus-Length-Constraint]]).
  - The radio network grows from ~9 nodes to over a hundred, which forces scheduled TDMA at tier 2 ([[Slave-Cell-Capacity-And-Scheduling]]).
  - Per-light fault identity comes from per-pole radio reports.
- **"1 is backup"** on the master's two radios → primary plus exercised standby ([[Dual-Radio-Redundancy-And-Coexistence]]).
- **"2 for efficiency"** on the slave's downstream radios → both active, on separate channels, sharing the devices.

## Not stated anywhere

- Physical distances between master, slaves and poles ([[Open-Questions]] Q16)
- Exact number of sections and devices ([[Open-Questions]] Q27)
- Which exact STM32 development board is supplied. Spec 06 proposes NUCLEO-L476RG for the gateways and asks for 3 boards + 10 radios on the bench from week 2

The radio band is now decided separately — see [[Philippines-Regulatory-Context]].

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Team-And-Ownership]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Open-Questions]]

## Sources

- `Sources/system-architecture-diagram-2026-09-09.jpg` — original issued 2026-09-09; re-issued with client annotations 2026-10-14 (same filename): "modbus" on the RS485 link, "and control" under Central Monitoring, "1 is backup / 2 LoRa radio" at the master, "4 LoRa" and "2 for efficiency (50 devices)" at slave 1.
- Verbal scope from project owner, 2026-09-09: STM32 for the board, custom (non-LoRaWAN) LoRa protocol.
- Verbal scope from project owner, 2026-10-14: Modbus; dual radios; SX1276 + STM32; one device per pole; one radio per device; section count may be 4 or more.
- `Specs/specs/04-channel-plan-and-airtime.md`, `Specs/specs/06-gateway-hardware-requirements.md` (v0.1).

## History

- 2026-09-09 — original diagram: no RS485 protocol named, one radio per gateway, 2 devices per section, ~9 radio nodes.
- 2026-10-14 — client annotations: Modbus, dual radios (4 per slave), ~50 devices per slave; per-pole devices and one radio per device confirmed; section count left open.
- 2026-10-21: aligned with spec pack v0.1 (pointer to the M1 pack, 120 s cycle, proposed dev board and bench set).

Back to [[Index]]
