# Modbus Register Map Design

The register map is the contract between the Master Gateway and the Local Monitoring Server over RS485 / Modbus RTU. The bottom line: a flat, sparse, fixed-stride map with a per-section stride, generated from one machine-readable source, where every value carries its age and quality.

## Key points

- **Generate the map from one source of truth** (CSV/JSON) that produces both the firmware headers and the server's poller config. Both ends are now pgorospe's ([[Local-Monitoring-Server-Design]]), which makes this easier, not optional.
- **Flat, sparse, fixed-stride**, with a **section stride**: the number of Slave Gateways (sections) isn't fixed, and each section holds up to 60 devices.
- **Every device block carries `age_seconds` and an `online` bit.** Never serve old data that looks fresh ([[Cache-Behind-Synchronous-Wire-Pattern]]).
- **Commands go through an atomic mailbox** with a magic trigger value, answered immediately and tracked as a job ([[Asynchronous-Command-Job-Pattern]]).
- **Use exceptions `0x0A` / `0x0B`.** They exist for gateways.

## Protocol facts that constrain the map

- Frame: `[addr 1B][function 1B][data 0–252B][CRC16 2B]`. Max ADU 256 B, max PDU 253 B.
- Addresses: 0 = broadcast (no reply), **1–247 valid**, 248–255 reserved.
- **CRC-16:** poly 0xA001 (reflected 0x8005), init 0xFFFF, no final XOR, and **transmitted low byte first**, the opposite of every other multi-byte field in Modbus. The STM32 hardware CRC unit can compute it with input/output bit-reversal configured.

| FC | Name | Limit |
|---|---|---|
| 0x03 | Read Holding Registers | **125 regs** |
| 0x04 | Read Input Registers | **125 regs** |
| 0x10 | Write Multiple Registers | **123 regs** |
| 0x17 | Read/Write Multiple | read 125, write 121 |

Exceptions: `01` Illegal Function, `02` Illegal Data Address, `03` Illegal Data Value, `04` Server Device Failure, `05` Acknowledge, `06` **Server Device Busy**, `0A` Gateway Path Unavailable, `0B` **Gateway Target Device Failed To Respond**.

## Layout: flat, sparse, fixed-stride

**Don't use a "device select + window" paging pattern.** It needs two transactions per read, isn't atomic (a retry or a second master corrupts the selection), and wastes a poll on selection.

```
address = base + (section × SECTION_STRIDE) + (instance × DEVICE_STRIDE) + point_offset
```

Sparse and wasteful, **on purpose**: computable by hand, stable across firmware versions, and nothing moves when a device or a section is added. Packed maps break every integrator on day one.

Proposed layout (to be frozen in the M1 specification):

```
INPUT REGISTERS (FC04) — read-only telemetry
  30001–30063   Gateway self/system: proto_version, fw_version, boot_count,
                uptime(32), unix_time(32), reset_cause, section_count,
                device_count_configured, device_count_online, per-radio
                tx_ok/tx_fail/rx_ok/crc_err, active radio / failover state,
                modbus frame/crc_err/exception counts, cpu_temp, vdd_mv,
                fault_bitmap(32), telemetry_epoch
  30101–30299   Slave Gateway blocks: one 16-register block per section
  30301–30499   Energy meter blocks: one 16-register block per section
  30501–30999   Summary block: 1 register per device (packed status + level)
  31001–…       Device blocks: section s at 31001 + s × 1000,
                device i at + i × 16  (60 devices × 16 = 960 regs per section)
  39000–39099   Diagnostics / commissioning / debug

HOLDING REGISTERS (FC03/06/10)
  40001–40031   Gateway config: baud, unit_id, RF channel plan / SF / power,
                cycle period, dusk/dawn offsets, lat/long ×10000, TZ
  40101–40140   COMMAND MAILBOX
  41001–…       Per-device setpoint shadow: section s at 41001 + s × 200,
                device i at + i × 2 (target_level, flags)
```

⚠ With a 1000-register section stride, 5-digit `3xxxx` notation fits about 8 sections (31001–38960). Beyond that, the server must use 0-based protocol addresses rather than `3xxxx` numbering. Decide the maximum section count at M1.

Per-device 16-register block, ~10 used and 6 reserved (**reserve generously; you will add fields**):

```
+0 present/status bitfield   +1 age_seconds        +2 level_actual (DALI 0–254)
+3 level_commanded           +4 lamp_current_mA    +5 lamp_voltage_dV
+6 power_W_x10               +7/+8 energy_Wh(32)   +9 temp_c
+10/+11 lamp_hours(32)       +12 fault_code        +13 fault_count
+14 last_rssi                +15 quality
```

## Staleness is a first-class field

Every device block carries `age_seconds` and an `online` bit. **A gateway that returns a last-known value with no indication of age is how SCADA systems act on hours-old data.**

Configurable policy: (a) return the cached value plus its age, **the default and most informative**; (b) return a `0xFFFF` sentinel when stale beyond N seconds; (c) return exception `0x0B` for the affected block. Because the server is in-house, (a) is the expected mode.

`telemetry_epoch` lets the server poll **one register** to detect "has anything changed", then bulk-read only what changed.

## Command mailbox

One atomic `FC16` write to a contiguous block. Full semantics in [[Asynchronous-Command-Job-Pattern]].

```
40101  cmd_sequence     (server increments; gateway echoes) — idempotency key
40102  cmd_opcode       1=set_level 2=group_set 3=identify 4=query_full
                        5=set_time 6=reboot_slave 7=start_ota 8=dali_raw
40103  cmd_target       device (section<<8 | instance) | 0xFFFF = all
                        | 0x8000|g = group g | section target   ⚠ encoding to freeze at M1
40104..40119  cmd_params[16]
40120  cmd_trigger      write 0xA5C3 to execute   ← guards against torn writes
--- read back ---
40121  ack_sequence     40122 cmd_state    40123 cmd_result
40124..40131  cmd_response[8]
```

Rules:
- Reject a new command while one is in flight with **exception `0x06`**.
- The trigger magic value means a server that crashes mid-`FC16` can't launch a half-formed command.
- **Never block the RS485 loop on a radio round trip:** respond `queued` immediately and let the server poll the job.

## Accommodations worth keeping

1. **FC03 mirror.** Many Modbus tools only implement FC03. Mirror the input-register space at `4xxxx = 3xxxx + 10000` behind a config flag. The in-house server doesn't need it, but it saves time when debugging with QModMaster or Modpoll.
2. **Summary block:** one register per device (packed status + level), so the server polls fast and deep-reads only what changed. 60 devices per section is one transaction per section.

## Poll-cycle arithmetic

Per section, the full device area is 60 × 16 = **960 registers** → ⌈960/125⌉ = **8 transactions**. At 19200 baud each ~260-byte exchange is ~150 ms of wire plus t3.5 plus turnaround ≈ 250 ms, so **~2 s per section** for a full sweep, or ~0.5 s at 115200. In normal operation the server reads the summary block and `telemetry_epoch`, and deep-reads only changed devices.

## Also expose

A **register-map version register**, and a **heartbeat/counter register** the server watches to detect a frozen gateway. Document scaling explicitly, and 32-bit values as register pairs **with the word order written down**.

## Related

- [[Local-Monitoring-Server-Design]]
- [[Northbound-Interface-Options]]
- [[RS485-Hardware-Design]]
- [[Cache-Behind-Synchronous-Wire-Pattern]]
- [[Asynchronous-Command-Job-Pattern]]

## Sources

- Research synthesis, 2026-09-09. Modbus Application Protocol V1.1b3: https://web.eecs.umich.edu/~modbus/documents/ModbusApplicationProtocol_v1_1.pdf · gateway stale-data strategies: https://docs.chipkin.com/protocols/modbus/

## History

- 2026-09-09: first layout, sized for 3 slaves, 3 meters and 128 luminaires.
- 2026-10-14: re-laid out with a per-section stride for an unfixed number of sections (up to 60 devices each). Noted that the server is now in-house, and added per-radio and failover registers.

Back to [[Index]]
