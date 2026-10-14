# Team And Ownership

Who owns what on the LoRa smart street lighting project. pgorospe owns everything on the infrastructure side; one developer owns the field devices and another owns the gateway hardware.

## Key points

- **pgorospe** owns the Local Monitoring Server, the Master Gateway, the Slave Gateway firmware and the LoRa protocol at both tiers.
- **Device developer** owns the LoRa Energy Meter and the per-pole LoRa LED Driver & Monitoring Device (Blue Pill STM32F103 + SX1276, one radio each).
- **Gateway hardware developer** owns the gateway PCBs, enclosures, power and surge protection, following the client-supplied STM32 dev boards and SX1276 modules (NUCLEO-L476RG recommended for gateways).
- The one interface seam that matters most is the **tier-2 radio protocol** between Slave Gateways and devices. pgorospe supplies both the spec and a linkable device-role library.
- There is no co-developer for the protocol library: pgorospe writes all roles.

## Role assignments

| Component | Owner |
|---|---|
| Local Monitoring Server | pgorospe |
| Master LoRa Gateway (1 unit, 2 radios) | pgorospe |
| Slave LoRa Gateway firmware (one per section, count TBC; 4 radios each) | pgorospe |
| LoRa protocol — spec and library, all roles | pgorospe |
| LoRa Energy Meter (one per section) | Device developer |
| LoRa LED Driver & Monitoring Device (one per pole) | Device developer |
| Gateway hardware (PCB, enclosure, power, surge, antennas) | Gateway hardware developer |

## The device-side seam

Everything from the server down to the Slave Gateways is one person's design, so the old negotiation-heavy interfaces between server, master and slave are gone. The boundary that still needs discipline is the slave ↔ device link:

1. **A written tier-2 wire-protocol spec**, frozen at the first milestone and signed by the device developer.
2. **A portable C device-role library** (`mac_role = DEVICE`) from the same repository as the master and slave roles, so the device developer *links* the protocol rather than re-implementing it. It must fit the Blue Pill: ≤16 KB flash, ≤3 KB RAM, bare-metal, no FPU — see [[STM32-Part-Selection]].
3. **Shared test vectors** (frame encodings, MIC, airtime) in CI on both sides.
4. **The simulator works both directions**: the fake device used to test the slave is also the fake slave the device developer tests against.

**Dependency risk:** fail-safe layer 3 — the device holding its own fallback schedule and the DALI `SYSTEM FAILURE LEVEL` — lives in firmware pgorospe does not write. **Get the fail-safe behaviour table signed by the device developer.** See [[Fail-Safe-Lighting-Rule]].

## Interface contracts pgorospe owns

| Contract | Signed with |
|---|---|
| Modbus register map ([[Modbus-Register-Map-Design]]) | Internal, but still versioned and machine-generated |
| Tier-1 protocol (master ↔ slaves) | Internal |
| Tier-2 protocol (slaves ↔ devices) | Device developer |
| Fail-safe behaviour table | Device developer and client |
| Gateway hardware requirements (radio wiring, independent SPI bus / antenna / coax per radio of a redundant pair, RS485, RTC backup) | Gateway hardware developer |

Generate the register map and wire protocol from **one machine-readable source of truth** so the two sides cannot drift.

## Programme

Contracted scope, milestones and terms are in the Statement of Work in `Output/`.

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Local-Monitoring-Server-Design]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Development-Roadmap-And-Gates]]

## Sources

- Project owner, 2026-09-13: four-person team, one per component.
- Project owner, 2026-10-14 (from client): pgorospe takes the server, master and slave gateways. One other person takes the energy meter and the LED driver/monitoring device. STM32 boards and SX1276 radios throughout. Devices on Blue Pill; separate gateway hardware developer.

## History

- 2026-09-13 — four developers, one per component (master, slave, server, hardware); MAC library co-developed with a slave developer.
- 2026-10-14 — pgorospe takes server, master and slave firmware plus the whole protocol; device developer and gateway hardware developer assigned; devices confirmed on Blue Pill with one radio each.

Back to [[Index]]
