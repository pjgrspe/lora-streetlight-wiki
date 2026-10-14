# Cache Behind a Synchronous Wire (Pattern)

The master gateway sits between a **fast synchronous wired protocol** (Modbus RTU over RS485, ~300–1000 ms timeouts) and a **slow half-duplex radio** (a two-hop LoRa round trip is ~1–1.5 s at SF9, 3–6 s at SF12). The mismatch is one to two orders of magnitude.

**Rule: never block a Modbus response on a LoRa transaction.** Answer every read in under 10 ms from RAM, always.

## Why blocking fails

- Modbus masters time out in 300–1000 ms with 2–3 retries; a LoRa round trip exceeds that by 1–2 orders of magnitude.
- Retries amplify radio traffic, causing self-inflicted congestion — the failure feeds itself.
- A slow reply arriving after the master moved on causes cascading RS485 frame misalignment: the master's buffer misreads the late response as a new frame.

## The pattern: value + freshness + quality

Every telemetry point is a **triple**, not a value:

```
value (16/32-bit)  |  timestamp / age  |  quality bitfield
```

| Field | Width | Semantics |
|---|---|---|
| `last_update_unix` | 32-bit | UTC seconds, stamped at origin (endpoint if it has an RTC, else the slave) |
| `age_seconds` | 16-bit | Convenience for dumb HMIs; saturates at 65535 |
| `quality` | 16-bit | bit0 STALE, bit1 NEVER_READ, bit2 SUBSTITUTED, bit3 COMM_FAIL, bit4 DEVICE_ALARM, bit5 PARENT_UNREACHABLE |

## The four rules

1. **Never hold a response.** Serve from cache unconditionally.
2. **Auto-stale.** A value older than `2 x expected_refresh_interval` flips to STALE — enforced by the *master*, not the server, because the server may turn out to be a spreadsheet macro.
3. **Never zero a stale value.** Zero is a legitimate reading (0 W = lamp off). Zeroing on comms loss manufactures phantom "all lights off" alarms. Keep the value, degrade the quality.
4. **Also publish a primitive signal.** A per-device `comm_ok` coil and a gateway-level `network_health` word, so a server that ignores quality bitfields can still alarm.

The governing principle, worth quoting in the spec: *preserve the last good value only with an explicitly stale/bad quality state; never make old data look fresh.*

## Prior art doing exactly this

- **LoRaBus** (sfera-labs) — a gateway on RS-485 representing remote LoRa nodes as Modbus registers. Same topology as ours.
- **Dragino MS48-LR** — writes LoRaWAN uplink data into Modbus FC3 registers; cache-as-register-image, no synchronous proxying.
- **Sparkplug B** — birth/death certificates are the modern equivalent of the quality flag.

## Related

- [[Asynchronous-Command-Job-Pattern]]
- [[Modbus-Register-Map-Design]]
- [[Master-LoRa-Gateway-Role]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://flowfuse.com/blog/2026/04/modbus-polling-best-practices/ · https://github.com/sfera-labs/lorabus-arduino
