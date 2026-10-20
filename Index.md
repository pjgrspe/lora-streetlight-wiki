# Index

Knowledge base for a LoRa smart street lighting system in the Philippines: a **Local Monitoring Server**, a **Master LoRa Gateway**, one **Slave LoRa Gateway per section**, and per-pole **LoRa LED driver / monitoring devices** that control the lights over DALI.

## The system in one picture

```
Local Monitoring Server ──RS485 / Modbus RTU──► MASTER GATEWAY (2 radios: primary + standby)
                                                   │ LoRa tier 1
                               ┌───────────────────┼───────────────────┐
                          SLAVE GATEWAY       SLAVE GATEWAY       SLAVE GATEWAY …   one per section,
                          (4 radios)                                                count TBC
                               │ LoRa tier 2 — 2 channels, both active
                          up to 60 devices per section: per-pole LED driver/monitoring devices
                          + 1 energy meter ──DALI──► luminaire
                          (both tiers repeat on a 120 s cycle by default)
```

| Owner | Scope |
|---|---|
| **pgorospe** | Local Monitoring Server, Master Gateway, Slave Gateway firmware, the LoRa protocol (all tiers) |
| Device developer | LoRa energy meter, LoRa LED driver & monitoring device (Blue Pill STM32F103 + SX1276) |
| Hardware developer | Gateway hardware, built on STM32 dev boards + SX1276 modules |

## Specs

The signed-off contract documents (the M1 pack) live in the `Specs/` folder, repo `lora-streetlight-specs`: specs 00–07 plus the register-map CSV and a decision log. **The specs override the wiki.** The wiki explains why; the specs say exactly what. Code lives in `Code/`, repo `lora-streetlight`.

| Spec | Topic |
|---|---|
| 00 | Common frame format and conventions |
| 01 | Modbus register map |
| 02 / 03 | Tier-1 and tier-2 radio protocols |
| 04 | Channel plan and airtime budget |
| 05 | Fail-safe behaviour |
| 06 | Gateway hardware requirements |
| 07 | Server data model and requirements |

## Start here

New to the project? Read these five, in order:

1. [[System-Architecture-As-Diagrammed]]: what the client drew, and how it's interpreted
2. [[Master-LoRa-Gateway-Role]]: what the master does, and why it isn't "just a radio bridge"
3. [[Fail-Safe-Lighting-Rule]]: the rule behind every failure decision (when in doubt, lights ON)
4. [[Slave-Cell-Capacity-And-Scheduling]]: how one slave serves ~50 devices without a bottleneck
5. [[Open-Questions]]: what's still undecided

## 1 · Project

- [[Team-And-Ownership]]: who builds what, and the interface contracts between people
- [[System-Architecture-As-Diagrammed]]: the client's diagram and its annotations
- [[Open-Questions]]: open and resolved questions, ranked
- [[Development-Roadmap-And-Gates]]: phases with testable exit gates
- [[Philippines-Regulatory-Context]]: the radio band and limits we design to (915–918 MHz)
- [[Philippines-Site-Context]]: grid, billing, climate

## 2 · Architecture

- [[Master-LoRa-Gateway-Role]]: the master's responsibilities and non-responsibilities
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]: why slaves run their own schedules
- [[Fail-Safe-Lighting-Rule]]: five independent layers of "the road stays lit"
- [[Cache-Behind-Synchronous-Wire-Pattern]]: never block a Modbus read on a radio round trip
- [[Asynchronous-Command-Job-Pattern]]: how commands work over a slow radio
- [[Popcorn-Effect-And-Scheduled-Execution]]: switching every light at the same instant
- [[Device-Identity-Namespaces]]: hardware ID vs network address vs pole ID
- [[Failure-Modes-And-Mitigations]]: the failure table (FMEA)
- [[LoRa-For-Control-Critique]]: the honest case against LoRa, and how this design answers it

## 3 · Radio and protocol

- [[LoRa-PHY-Fundamentals]]: spreading factor, bandwidth, the basics
- [[LoRa-Airtime-Budget]]: time on air, the 400 ms limit, duty cycle
- [[Link-Budget-And-Site-Survey]]: realistic range and how to survey
- [[LoRa-vs-LoRaWAN]]: why a custom protocol
- [[Custom-Protocol-Frame-Design]]: addressing, frame format, ACK and retry
- [[Media-Access-Control-Options]]: TDMA vs ALOHA vs CAD
- [[Slave-Cell-Capacity-And-Scheduling]]: the time-slotted device network per section
- [[Dual-Radio-Redundancy-And-Coexistence]]: what paired radios buy, and why radios in one box blind each other
- [[Continuous-Receive-Downlink]]: why mains-powered nodes listen all the time
- [[Custom-Protocol-Security]]: message authentication, replay protection, keys

## 4 · Gateway hardware

- [[STM32-Part-Selection]]: gateway board (NUCLEO-L476RG) and device platform (Blue Pill) constraints
- [[Radio-Hardware-Selection]]: SX1276 modules, antennas, supply risk
- [[RS485-Hardware-Design]]: isolation, termination, bias, timing
- [[Enclosure-Power-And-Surge]]: lightning, humidity, coax loss

## 5 · Gateway firmware

- [[Firmware-Architecture]]: RTOS tasks, the receive path, layering
- [[Modbus-Register-Map-Design]]: the contract between gateway and server
- [[Persistence-And-Storage]]: flash traps, FRAM, journalling
- [[Time-Distribution-And-Clock-Trust]]: RTC, beacons, the astronomical clock
- [[Watchdog-And-Fault-Recovery]]: watchdog discipline and recovery
- [[Firmware-Update-Over-LoRa]]: update paths and their limits
- [[Gateway-Observability]]: what a field technician can see
- [[Development-Tooling]]: build, test, debug, bench fixture

## 6 · Local Monitoring Server

- [[Local-Monitoring-Server-Design]]: poller, database, dashboard, control
- [[Northbound-Interface-Options]]: Modbus now; MQTT/TALQ later as bridges

## 7 · Devices and DALI

- [[DALI-Fundamentals]]: the bus, addressing, commands, timing
- [[DALI-Bus-Length-Constraint]]: why the system went per-pole
- [[DALI-Telemetry-And-D4i]]: what existing drivers can and can't report
- [[Energy-Metering-And-Telemetry-Contract]]: what to measure, and fault detection
- [[Street-Lighting-Control-Logic]]: astronomical schedules, dimming, savings
- [[Commissioning-And-Field-Swap]]: installing, mapping and replacing devices

## How notes are written

- One topic per note. Each opens with a short summary and **Key points**, then detail, **Related**, **Sources**, and a short **History** of what changed.
- Notes describe the **current** design. Earlier decisions are kept as one-line History entries, not as banners at the top.
- **⚠ marks a claim that isn't verified yet.** Treat it as a work item.
- Commercial terms (price, hours) live in the Statement of Work in `Output/`, not in this wiki.
