# Team And Ownership

Who owns what on the LoRa smart street lighting project. pgorospe owns everything on the infrastructure side; one developer owns the field devices and another owns the gateway hardware.

## Key points

- **pgorospe** owns the Local Monitoring Server, the Master Gateway, the Slave Gateway firmware and the LoRa protocol at both tiers.
- **Device developer** owns the LoRa Energy Meter and the per-pole LoRa LED Driver & Monitoring Device (Blue Pill STM32F103 + SX1276, one radio each).
- **Gateway hardware developer** owns the gateway PCBs, enclosures, power and surge protection, following the client-supplied STM32 dev boards and SX1276 modules (NUCLEO-L476RG recommended for gateways). **Also the client's technical decision-maker and pgorospe's route to the client**: the client isn't involved in technical detail.
- The interface contracts are written down as **specs 00–07** in the `lora-streetlight-specs` repo (the M1 pack). Each party signs the specs that bind them.
- The one interface seam that matters most is the **tier-2 radio protocol** (spec 03) between Slave Gateways and devices. pgorospe supplies both the spec and a linkable device-role library.
- There is no co-developer for the protocol library: pgorospe writes all roles. Code lives in the `lora-streetlight` repo.

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

1. **A written tier-2 wire-protocol spec** (spec 03), frozen at M1 and signed by the device developer. Its §13 lists what the device firmware must do around the library.
2. **A portable C device-role library** (`mac_role = DEVICE`) from the same repository as the master and slave roles, so the device developer *links* the protocol rather than re-implementing it. It must fit the Blue Pill: ≤16 KB flash, ≤3 KB RAM, bare-metal, no FPU — see [[STM32-Part-Selection]].
3. **Shared test vectors** (frame encodings, MIC, airtime), delivered with the library at M4 and run in CI on both sides.
4. **The simulator works both directions**: the fake device used to test the slave is also the fake slave the device developer tests against.

**Dependency risk:** fail-safe layer 3 — the device holding its own fallback schedule and the DALI `SYSTEM FAILURE LEVEL` — lives in firmware pgorospe does not write. Spec 05 turns this into numbered, testable device obligations (DD-1…DD-11), and the device developer signs it. See [[Fail-Safe-Lighting-Rule]].

## The spec pack and who signs it

All specs are in `Specs/specs/` (v0.1 drafts as of 2026-10-21).

| Spec | Contract | Signed by |
|---|---|---|
| 00 | Common frame format, addressing, security layer | All parties |
| 01 | Modbus register map + CSV source ([[Modbus-Register-Map-Design]]) | Hardware developer (for the client) |
| 02 | Tier-1 protocol (master ↔ slaves) | Hardware developer (for the client) |
| 03 | Tier-2 protocol (slaves ↔ devices) | **Device developer**, hardware developer (for the client) |
| 04 | Channel plan and airtime budget | Hardware developer (for the client) |
| 05 | Fail-safe behaviour table | **Hardware developer (for the client) and device developer** |
| 06 | Gateway hardware requirements | **Gateway hardware developer** |
| 07 | Server data model, dashboard and control requirements | Hardware developer (for the client) |

The register map is generated from **one machine-readable source of truth** (`Specs/register-map/register-map.csv`) so the firmware and server can't drift.

## How the team aligns

- **Communication:** pgorospe talks only to the two developers, over Messenger. Client items (purchasing, the NTC letter, tariff and pole-count questions) go through the hardware developer.
- **Review in rounds.** Each draft round is tagged (`review-r1`…). Comments come in as GitHub issues, one per question, within a 3-working-day window. One walkthrough call per round decides them, and decisions go in `Specs/DECISIONS.md`.
- **Sign-off** on the tag `spec-v1.0`: each party approves in writing (a Messenger message is fine) or signs the PDF sign-off page. That completes M1.
- **After sign-off,** a spec changes only through a change request (a GitHub issue labelled `change-request`) with its effort impact, approved in writing.

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
- `Specs/README.md` and `Specs/specs/00`–`07` (v0.1, 2026-10-21).

## History

- 2026-09-13 — four developers, one per component (master, slave, server, hardware); MAC library co-developed with a slave developer.
- 2026-10-14 — pgorospe takes server, master and slave firmware plus the whole protocol; device developer and gateway hardware developer assigned; devices confirmed on Blue Pill with one radio each.
- 2026-10-21: aligned with spec pack v0.1 (contracts now specs 00–07 with named signers; review, sign-off and change-request process; specs and code repos).
- 2026-10-21: the hardware developer decides technical matters for the client and signs the client's specs; communication with the team is over Messenger.

Back to [[Index]]
