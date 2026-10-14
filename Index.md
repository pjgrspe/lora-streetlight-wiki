# Index

Master index for the LORA wiki, a knowledge base for the 128-light LoRa smart street lighting system (Philippines). **Since 2026-10-14, pgorospe's scope is the Local Monitoring Server, the Master Gateway and the three Slave Gateways**, plus the LoRa protocol at both tiers. The device side (energy meter, LED driver/monitoring device) belongs to one other developer. Every topic note links back here.

> **2026-10-14 revision:** dual radios master↔slave, 4 radios per slave, ~50 devices per slave (apparently per-pole), Modbus on RS485, SX1276 on STM32. Start with [[System-Architecture-As-Diagrammed]] → [[Dual-Radio-Redundancy-And-Coexistence]] → [[Slave-Cell-Capacity-And-Scheduling]]. Notes predating this carry an **Updated/Superseded 2026-10-14** callout at the top.

**Start with [[Master-LoRa-Gateway-Role]]** if you're new to the project, or **[[Open-Questions]]** if you want to know what's still undecided.

## How to use this vault

- One atomic concept per note — no "everything about X" notes.
- Link liberally, including to notes that don't exist yet — a red `[[link]]` marks something worth writing later.
- When new material supersedes a note, add a **Superseded** callout with the date and what changed. Don't silently overwrite history.
- Note filenames are `Title-Case-With-Hyphens.md`, named after the concept, not the source it came from.

## Project scope

- [[Team-And-Ownership]] — who owns what; pgorospe owns the server, master and slaves (rev 2026-10-14)
- [[System-Architecture-As-Diagrammed]] — the literal contents of the architecture diagram, separated from interpretation
- [[Master-LoRa-Gateway-Role]] — what this box is responsible for. **The single most important note.**
- [[Open-Questions]] — everything unresolved, ranked. **Q1–Q5 could force a redesign.**
- [[Development-Roadmap-And-Gates]] — phased plan with testable exit gates

## Core architecture decisions

- [[Cache-Behind-Synchronous-Wire-Pattern]] — never block a Modbus read on a LoRa round trip
- [[Asynchronous-Command-Job-Pattern]] — how commands work when they can't complete in a timeout
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]] — why the slaves get their own brains
- [[Fail-Safe-Lighting-Rule]] — ambiguous failure resolves to LIGHTS ON
- [[Popcorn-Effect-And-Scheduled-Execution]] — multicast + "execute at time T"
- [[Device-Identity-Namespaces]] — UID vs logical address vs asset ID
- [[Failure-Modes-And-Mitigations]] — FMEA, 15 failure modes
- [[LoRa-For-Control-Critique]] — the honest case against LoRa here, and why this design survives it
- [[Dual-Radio-Redundancy-And-Coexistence]] — what the paired radios buy, and why radios in one box blind each other (new 2026-10-14)
- [[Slave-Cell-Capacity-And-Scheduling]] — serving ~54 devices per slave: scheduled uplink, not polling (new 2026-10-14)

## Radio and protocol

- [[LoRa-PHY-Fundamentals]] — CSS, spreading factor, coding rate, the header/CRC traps
- [[LoRa-Airtime-Budget]] — time-on-air tables, the 400 ms dwell problem, the duty-cycle conflict
- [[Link-Budget-And-Site-Survey]] — realistic ranges, Fresnel, and the survey protocol
- [[LoRa-vs-LoRaWAN]] — what a custom MAC costs you, and when it's justified
- [[Custom-Protocol-Frame-Design]] — addressing, frame format, ACK/retry, fragmentation
- [[Media-Access-Control-Options]] — TDMA vs ALOHA vs CAD, and the recommended superframe
- [[Custom-Protocol-Security]] — AES-CTR + CMAC, and the ten things custom protocols get wrong
- [[Continuous-Receive-Downlink]] — why permanent receive is free on a mains-powered node
- [[Philippines-Regulatory-Context]] — NTC, 915–918 MHz, and the biggest open risk

## Gateway hardware

- [[STM32-Part-Selection]] — STM32WLE5 vs the G0B1 fallback; flash/RAM budget. **Now: common STM32 + external SX1276 board (2026-10-14)**
- [[Radio-Hardware-Selection]] — SX1262 vs SX1276, TCXO, modules vs bare chip. **Client chose SX1276 (2026-10-14)**
- [[RS485-Hardware-Design]] — isolation, termination, fail-safe bias, DE timing
- [[Enclosure-Power-And-Surge]] — mains, lightning, humidity, coax loss

## Gateway firmware

- [[Firmware-Architecture]] — FreeRTOS task decomposition, the DMA receive path, layering
- [[Modbus-Register-Map-Design]] — the contract with the monitoring server
- [[Persistence-And-Storage]] — the flash ECC trap, FRAM, LittleFS
- [[Time-Distribution-And-Clock-Trust]] — RTC, beacons, and the local astronomical clock
- [[Watchdog-And-Fault-Recovery]] — IWDG discipline, crash capture, escalation ladder
- [[Firmware-Update-Over-LoRa]] — NOR staging, signing, and whether OTA is feasible
- [[Gateway-Observability]] — what a field technician needs to see
- [[Development-Tooling]] — build, test, HIL fixture, logging

## Downstream domain

- [[DALI-Fundamentals]] — bus, addressing, commands, the dimming curve, timing limits
- [[DALI-Bus-Length-Constraint]] — why one bus per section can't work, and the fork it creates
- [[DALI-Telemetry-And-D4i]] — Parts 251/252/253, and the D4i procurement decision
- [[Energy-Metering-And-Telemetry-Contract]] — what to measure and the three fault-detection layers
- [[Street-Lighting-Control-Logic]] — astro clock, dimming profiles, and measured vs modelled savings
- [[Northbound-Interface-Options]] — Modbus, MQTT, TALQ, and the server stack

## Local Monitoring Server

- [[Local-Monitoring-Server-Design]] — poller, historian, dashboard, control; recommended stack (new 2026-10-14)

## Operations

- [[Commissioning-And-Field-Swap]] — join model, site survey, the 10-minute device swap
- [[Philippines-Site-Context]] — grid, billing model, environment

## Conventions

Every note opens with a one-line definition, then detail, then `## Related` wikilinks, then a link back here. Source attribution goes in a `## Sources` section at the bottom with the date, so a claim can be traced. **Uncertainty is flagged inline with ⚠ rather than smoothed over** — a lot of this project is still unknown, and the notes should say so.
