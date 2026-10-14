# Team And Ownership

Who owns what on the LoRa smart street lighting project.

> **Superseded 2026-10-14.** The original split (recorded 2026-09-13) was four developers, one per component: master (pgorospe), slave, server, hardware. The shared MAC was co-developed with a slave gateway developer. **That no longer holds.** pgorospe now owns the server, the master *and* the slaves. One other developer owns the device side. The old split is kept at the bottom for history.

## Role assignments (current)

| Component | Owner |
|---|---|
| **Local Monitoring Server** | **pgorospe** |
| **Master LoRa Gateway** (×1, 2 radios) | **pgorospe** |
| **Slave LoRa Gateways** (×3, 4 radios each) | **pgorospe** |
| **LoRa protocol: all tiers, spec and gateway-side library** | **pgorospe** |
| LoRa Energy Meter (×3) | Device developer |
| LoRa LED Driver & Monitoring Device (per pole, ~128) | Device developer |
| Gateway hardware (PCB, enclosure, power, surge) | **Gateway hardware developer** (separate). Follows the client-supplied STM32 dev boards + SX1276 modules that pgorospe develops on, and pgorospe's hardware requirements spec |

## What this means

- **The whole infrastructure side is one person's design**: server → Modbus → master → LoRa tier 1 → slaves → LoRa tier 2. That removes three of the interface negotiations that used to be the project's biggest seams.
- **One interface seam remains, and it's the important one: the tier-2 radio protocol between the slaves and the devices.** pgorospe defines it. The device developer implements the device side. Make that boundary concrete:
  1. **A written tier-2 wire-protocol spec**, frozen at M1 and signed by the device developer.
  2. **A portable C device-role MAC library** (`mac_role = DEVICE`) from the same repository as the gateway roles, so the device developer *links* the protocol rather than re-implementing it from the spec. **Devices run on Blue Pill (STM32F103C8) + SX1276**, while the gateways use a Nucleo-64-class board. So the library must fit ≤16 KB flash / ≤3 KB RAM, bare-metal, no FPU. See [[STM32-Part-Selection]].
  3. **Shared test vectors** (frame encodings, MIC, airtime) in CI on both sides.
  4. **The simulator works both directions**: the fake device used to test the slave is also the fake slave the device developer tests against.
- **Fail-safe layer 3** (the device holding its own fallback schedule and the DALI `SYSTEM FAILURE LEVEL`) still lives in firmware pgorospe doesn't write. **Get the fail-safe behaviour table signed by the device developer.** See [[Fail-Safe-Lighting-Rule]].
- **There is no longer a co-developer for the MAC library.** The old SOW assumption A5 ("+60–90 h if sole development") has now come true, and it's priced into the v5 quotation.

## Interface contracts pgorospe owns

1. **Modbus register map** (internal now, but still versioned and machine-generated: [[Modbus-Register-Map-Design]])
2. **LoRa tier-1 protocol**: master ↔ slaves (internal)
3. **LoRa tier-2 protocol**: slaves ↔ devices → **signed with the device developer**
4. **Fail-safe behaviour table** → signed with the device developer and the client
5. **Gateway hardware requirements** → signed with the gateway hardware developer (radio wiring, independent SPI/antenna/coax per radio of a redundant pair)

## Programme

See `Output/LoRa-System-Statement-of-Work.md` (v5, 2026-10-14) for the contracted scope and the re-assessed quotation.

## History: the 2026-09-13 split (superseded)

| Component | Owner then |
|---|---|
| Master LoRa Gateway | pgorospe |
| Slave LoRa Gateway | Slave gateway developer |
| Local Monitoring Server | Server developer |
| Hardware | Hardware developer |

The MAC was one library with `mac_role = MASTER | SLAVE`. pgorospe owned the core and the master role, and the slave developer owned the slave role. The slot structure, beacon format and time-sync maths were designed jointly. The master engagement was quoted at 150 h (SOW v4).

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Local-Monitoring-Server-Design]]
- [[Slave-Gateways-As-Intelligent-Sub-Controllers]]
- [[Development-Roadmap-And-Gates]]
- [[Index]]

## Sources

- Project owner, 2026-09-13: four-person team, one per component.
- Project owner, 2026-10-14 (from client): pgorospe takes the server, master and slave gateways. One other person takes the energy meter and the LED driver/monitoring device. STM32 boards and SX1276 radios throughout.
