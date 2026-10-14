# Modbus Register Map Design

The register map is the contract between the master gateway and the Local Monitoring Server. It is also the thing an integrator will curse you for. Generate it from one machine-readable source of truth (CSV/JSON) that produces both firmware headers and server config — **hand-maintained duplicate maps are the number-one source of integration pain.**

## Protocol facts that constrain the map

- Frame: `[addr 1B][function 1B][data 0–252B][CRC16 2B]`. Max ADU 256 B, max PDU 253 B.
- Addresses: 0 = broadcast (no reply), **1–247 valid**, 248–255 reserved.
- **CRC-16**: poly 0xA001 (reflected 0x8005), init 0xFFFF, no final XOR, and **transmitted low byte first** — the opposite byte order from every other multi-byte field in Modbus. The STM32 hardware CRC unit can do it with bit-reversal configured; worth using.

| FC | Name | Limit |
|---|---|---|
| 0x03 | Read Holding Registers | **125 regs** |
| 0x04 | Read Input Registers | **125 regs** |
| 0x10 | Write Multiple Registers | **123 regs** |
| 0x17 | Read/Write Multiple | read 125, write 121 |

Exceptions: `01` Illegal Function, `02` Illegal Data Address, `03` Illegal Data Value, `04` Server Device Failure, `05` Acknowledge, `06` **Server Device Busy**, `0A` Gateway Path Unavailable, `0B` **Gateway Target Device Failed To Respond**.

> **`0x0A` and `0x0B` exist precisely for your use case. You are a gateway. Use them.**

## Layout: flat, sparse, fixed-stride

**Do not use a "device select + window" paging pattern.** It needs two transactions per read, is not atomic (two masters, or one master with retries, corrupt each other's selection), and burns a poll cycle on selection. Your ~134 devices fit without it.

Address = `base + (section × SECTION_STRIDE) + role_offset + (instance × DEVICE_STRIDE) + point_offset`. Sparse and wasteful — **that is the point.** It is computable by hand, stable across firmware versions, and doesn't shift when a device is added. Packed maps break every integrator on day one.

```
INPUT REGISTERS (FC04) — read-only telemetry
  30001–30063   Gateway self/system: proto_version, fw_version, boot_count,
                uptime(32), unix_time(32), reset_cause, device_count_configured,
                device_count_online, radio tx_ok/tx_fail/rx_ok/crc_err,
                modbus frame/crc_err/exception counts, last_rssi, last_snr,
                cpu_temp, vdd_mv, fault_bitmap(32), telemetry_epoch
  30101–30132   Slave gateway blocks: 3 × 16
  30201–30264   Energy meter blocks: 3 × 16
  31001–33048   Luminaire blocks: 128 × 16
  39000–39099   Diagnostics / commissioning / debug

HOLDING REGISTERS (FC03/06/10)
  40001–40031   Gateway config: baud, unit_id, RF channel/SF/BW/power,
                poll period, dusk/dawn offsets, lat/long ×10000, TZ
  40101–40140   COMMAND MAILBOX
  40201–40456   Per-device setpoint shadow: 128 × 2 (target_dim, flags)
```

Per-luminaire 16-register block, ~10 used and 6 reserved (**reserve generously — you will add fields**):

```
+0 present/status bitfield   +1 age_seconds        +2 level_actual (DALI 0–254)
+3 level_commanded           +4 lamp_current_mA    +5 lamp_voltage_dV
+6 power_W_x10               +7/+8 energy_Wh(32)   +9 temp_c
+10/+11 lamp_hours(32)       +12 fault_code        +13 fault_count
+14 last_rssi                +15 quality
```

## Staleness is a first-class field

Every device block carries `age_seconds` and an `online` bit. **A gateway that returns a last-known value with no indication of age is how SCADA systems act on hours-old data.** See [[Cache-Behind-Synchronous-Wire-Pattern]].

Offer a configurable policy: (a) always return cached value + age — **the default, most informative**; (b) return a `0xFFFF` sentinel when stale beyond N seconds; (c) return exception `0x0B` for the affected block.

`telemetry_epoch` lets an efficient server poll **one register** to detect "has anything changed", then bulk-read. Cheap and effective.

## Command mailbox

A single atomic `FC16` write to a contiguous block. Full semantics in [[Asynchronous-Command-Job-Pattern]].

```
40101  cmd_sequence     (master increments; gateway echoes)
40102  cmd_opcode       1=set_dim 2=group_set 3=identify 4=query_full
                        5=set_time 6=reboot_slave 7=start_ota 8=dali_raw
40103  cmd_target       device index | 0xFFFF=all | 0x8000|g = group g
40104..40119  cmd_params[16]
40120  cmd_trigger      write 0xA5C3 to execute   ← guards against torn writes
--- read back ---
40121  ack_sequence     40122 cmd_state    40123 cmd_result
40124..40131  cmd_response[8]
```

Rules: reject a new command while one is in flight with **exception `0x06`**; the trigger magic value means a master that crashes mid-`FC16` cannot launch a half-formed command; **never block the RS-485 loop on a radio round trip** — respond `queued` immediately and let the server poll.

## Two pragmatic accommodations

1. **A large fraction of Modbus masters only implement FC03.** Mirror the entire input-register space at holding-register addresses (`4xxxx = 3xxxx + 10000`) behind a config flag. Costs nothing, saves an integration week.
2. **Expose a compact summary block** — 128 × 1 register of packed status+dim, = 2 transactions — so the server polls fast and deep-reads only what changed.

## Poll-cycle arithmetic

The full luminaire block is 128 × 16 = **2048 registers** → ⌈2048/125⌉ = **17 transactions**. At 19200 baud each ~260-byte exchange is ~150 ms of wire plus t3.5 plus master turnaround ≈ 250 ms → **~4.3 s for a full sweep**. At 115200: ~1 s. Acceptable for street lighting.

## Also expose

A **register-map version register**, and a **heartbeat/counter register** the server can watch to detect a frozen gateway. Document scaling explicitly, and 32-bit values as consistent register pairs **with the word order written down**.

## Related

- [[Northbound-Interface-Options]]
- [[RS485-Hardware-Design]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Modbus Application Protocol V1.1b3: https://web.eecs.umich.edu/~modbus/documents/ModbusApplicationProtocol_v1_1.pdf · gateway stale-data strategies: https://docs.chipkin.com/protocols/modbus/
