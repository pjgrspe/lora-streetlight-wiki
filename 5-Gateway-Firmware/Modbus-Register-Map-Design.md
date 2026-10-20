# Modbus Register Map Design

The register map is the contract between the Master Gateway and the Local Monitoring Server over RS485 / Modbus RTU. The bottom line: a flat, sparse, fixed-stride map with a per-section stride, generated from one CSV, where every value carries its age and quality. The full map is specified in `Specs/specs/01-modbus-register-map.md` (v0.1); this note explains the reasoning.

## Key points

- **One source of truth:** `Specs/register-map/register-map.csv`. The firmware header and the server's poller config are generated from it, never typed by hand. If the spec text and the CSV disagree, the CSV wins.
- **0-based protocol addresses in hex.** No `3xxxx` / `4xxxx` notation, so there is no "fits ~8 sections" limit. Sections 1–14 all fit.
- **Flat, sparse, fixed-stride**, with a section stride. Nothing moves when a section or device is added. Unconfigured blocks read as zeros, not exceptions.
- **Every device block carries its age and a comm state** (online / stale / offline / not configured). Old data never looks fresh ([[Cache-Behind-Synchronous-Wire-Pattern]]).
- **Commands go through an atomic mailbox** (one FC16 write, CRC, magic trigger `0xA5C3`) and are tracked as jobs ([[Asynchronous-Command-Job-Pattern]]). Configuration is applied by one commit.
- **32-bit values are two registers, high word first.**

## Protocol facts that constrain the map

- Frame: `[addr 1B][function 1B][data 0–252B][CRC16 2B]`. Max ADU 256 B. FC03/FC04 read at most **125** registers, FC16 writes at most **123**.
- **CRC-16:** poly 0xA001 (reflected 0x8005), init 0xFFFF, transmitted low byte first.
- Serial default **19200 8E1**, unit ID 1. Above 19200 baud the inter-frame gap is fixed at 1.750 ms, not computed from character time ([[RS485-Hardware-Design]]).
- Function codes used: FC03, FC04, FC06, FC16. Exceptions used: `0x01`, `0x02` (outside the valid areas, or writes to read-only/reserved registers), `0x03`, `0x04`, `0x06` (job queue full or commit in progress), and `0x0B` only under the optional "stale = exception" policy. `0x0A` is not used.

## Layout

```
address = base + s × section_stride + n × device_stride + offset
          s = GwID (1–14), n = NodeID (1–63)
```

| Space | Area | Address |
|---|---|---|
| Input (FC04) | System / Master | `0x0000`–`0x00FF` |
| | Section (slave) blocks | `0x0100` + s × 16 |
| | Meter blocks | `0x0200` + s × 32 |
| | Summary, 1 register per device | `0x0400` + s × 64 + n |
| | Event window | `0x0800`–`0x090F` |
| | Device blocks, 16 registers each | `0x1000` + s × `0x400` + n × 16 |
| | Tier-1 diagnostics per slave | `0x7000` + s × 16 |
| Holding (FC03/06/16) | Configuration + commit | `0x0000`–`0x00FF` |
| | Command mailbox + job readback | `0x0100`–`0x013F` |
| | Fail masks per section | `0x0140` + s × 4 |
| | Job table (last 8 jobs) | `0x0180`–`0x01BF` |
| | Event acknowledge (`event_ack_seq`) | `0x0200` |
| | Optional FC03 mirror of the input space | `0x8000` + address |

Worked example: device 17 in section 3 (address `0x311`) has its block at `0x1D10`–`0x1D1F` and its summary register at `0x04D1`.

**Sparse on purpose:** computable by hand, stable across firmware versions, and nothing moves when the site grows. Packed maps break every integrator on day one. There is no per-device setpoint area: commands are jobs, and the commanded level is read back from the device block.

## Device block and summary register

Each device block (16 registers) holds the status bits, the age of the status, level actual and commanded, power, voltage, current, energy (32-bit), temperature, lamp hours (32-bit), fault code, RSSI/SNR, slot and home channel, and the age of the detail data. Status bits are the same as in the radio report (`Specs/specs/03-tier2-device-protocol.md` §6).

Two ages, because the data arrives at two rates: status every cycle (2 min), power and energy in the 15-minute detail records ([[Slave-Cell-Capacity-And-Scheduling]]).

The **summary register** packs one device into 16 bits: level (bits 0–7), comm state (bits 8–9), lamp on, fault, fallback, override, commanded off, config mismatch. A whole section's state is one 64-register read.

## Staleness is a first-class field

**A gateway that returns a last-known value with no indication of age is how SCADA systems act on hours-old data.**

Comm state: online while the status age is under the stale threshold (default 360 s = 3 cycles); stale beyond it; offline when the slave reports the device missing (3 missed reports). A configurable policy decides what stale blocks return: last value + age (default), "not available" sentinels, or exception `0x0B`.

`telemetry_epoch` lets the server poll **one register** to detect "has anything changed", then read only what changed. A `heartbeat` register lets the server detect a frozen Master.

## Command mailbox and jobs

One atomic FC16 write of `0x0100`–`0x0116`: `cmd_seq` (idempotency key), opcode, target, `execute_at`, 16 parameters, a CRC over the block, and the trigger `0xA5C3`. Any other write touching the block is rejected. The trigger and CRC mean a server that crashes mid-write can't launch a half-formed command. Opcodes and job states: [[Asynchronous-Command-Job-Pattern]].

**Never block the RS485 loop on a radio round trip:** the Master answers within 50 ms and the server follows the job.

## Events

The Master keeps a store-and-forward event buffer (at least 1000 events). The server reads the event window, stores each event durably, then acknowledges by writing `event_ack_seq`. Events are never dropped without an acknowledgement unless the buffer overflows (which raises its own event). Event code ranges: slave `0x01`–`0x3F`, device `0x40`–`0x7F`, Master `0x8000`+.

## Poll-cycle arithmetic

**Device blocks are read whole, at most 7 per request (112 registers)**, so no 32-bit value is split across two requests. A 60-device section is **9 requests**, about 1.3 s at 19200 baud (0.3 s at 115200). Routine polling (system head + event header every 2 s) uses about 4% of the bus. The server deep-reads only devices whose summary changed, and reads every block once per detail interval.

## Accommodations worth keeping

1. **FC03 mirror** of the input space at `0x8000` + address, on by default, so FC03-only tools (QModMaster, Modpoll) can read everything while debugging.
2. **Map version registers.** The server refuses to poll if the major version doesn't match the one it was generated for.

## Related

- [[Local-Monitoring-Server-Design]]
- [[Northbound-Interface-Options]]
- [[RS485-Hardware-Design]]
- [[Cache-Behind-Synchronous-Wire-Pattern]]
- [[Asynchronous-Command-Job-Pattern]]

## Sources

- `Specs/specs/01-modbus-register-map.md` and `Specs/register-map/register-map.csv` (v0.1, 2026-10-20)
- Research synthesis, 2026-09-09. Modbus Application Protocol V1.1b3: https://web.eecs.umich.edu/~modbus/documents/ModbusApplicationProtocol_v1_1.pdf · gateway stale-data strategies: https://docs.chipkin.com/protocols/modbus/

## History

- 2026-09-09: first layout, sized for 3 slaves, 3 meters and 128 luminaires.
- 2026-10-14: re-laid out with a per-section stride for an unfixed number of sections; per-radio and failover registers added.
- 2026-10-21: aligned with spec pack v0.1 (0-based hex addresses, new area layout, whole-block reads in 9 requests per section, event window, job table, setpoint shadow dropped).

Back to [[Index]]
