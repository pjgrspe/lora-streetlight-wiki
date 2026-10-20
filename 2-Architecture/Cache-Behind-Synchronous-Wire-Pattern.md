# Cache Behind a Synchronous Wire (Pattern)

The master sits between a **fast synchronous wire** (Modbus RTU over RS485, 300–1000 ms master timeouts) and a **slow half-duplex radio network** (device status arrives once per 120 s cycle; a single-pole command takes up to ~30 s). The rule that follows: **never block a Modbus response on a LoRa transaction. Answer every read from RAM, always.** The spec requires the Master to start every response within 50 ms (`Specs/specs/01-modbus-register-map.md` §2, v0.1).

## Key points

- The Modbus register image *is* the telemetry cache. Reads never touch the radio.
- Every block carries **value + age + comm state**.
- The master enforces staleness itself (default 360 s = 3 cycles), not the server.
- Never zero a stale value. Keep it and degrade its quality.
- A one-register-per-device summary gives a primitive comm state for simple tools.

## Why blocking fails

- Modbus masters time out in 300–1000 ms with 2–3 retries. A LoRa round trip exceeds that by one to two orders of magnitude.
- Retries amplify radio traffic, causing self-inflicted congestion that feeds itself.
- A slow reply that arrives after the server has moved on causes cascading RS485 frame misalignment: the server's buffer misreads the late response as a new frame.

## The pattern: value + freshness + quality

The general form:

```
value (16/32-bit)  |  timestamp / age  |  quality bitfield
```

| Field | Width | Semantics |
|---|---|---|
| `last_update_unix` | 32-bit | UTC seconds, stamped at origin (the device if it has an RTC, else the Slave) |
| `age_seconds` | 16-bit | Convenience for simple HMIs; saturates at 65535 |
| `quality` | 16-bit | bit0 STALE, bit1 NEVER_READ, bit2 SUBSTITUTED, bit3 COMM_FAIL, bit4 DEVICE_ALARM, bit5 PARENT_UNREACHABLE |

**As specified for this project (v0.1):** each device block has two ages, `age_status_s` (status, every cycle) and `age_detail_s` (power and energy, every 15 min), saturating at `0xFFFE`, with `0xFFFF` = never received. The summary register carries a 2-bit comm state: not configured / online / stale / offline. Section and meter blocks carry their own ages. A per-block timestamp and a quality bitfield were not needed once the server became in-house.

## The four rules

1. **Never hold a response.** Serve from cache unconditionally.
2. **Auto-stale.** A device whose status is older than the stale threshold (default 3 cycles) shows comm state STALE; one its slave reports missing shows OFFLINE. The *master* enforces this, so data is never presented fresh even if the server is naive.
3. **Never zero a stale value.** Zero is a legitimate reading (0 W = lamp off). Zeroing on comms loss manufactures phantom "all lights off" alarms.
4. **Publish a primitive signal too:** the per-device summary register, and a gateway `heartbeat` register the server watches to detect a frozen Master.

The governing principle, worth quoting in the spec: *preserve the last good value only with an explicitly stale/bad quality state; never make old data look fresh.*

Since 2026-10-14 the Local Monitoring Server is also pgorospe's deliverable ([[Local-Monitoring-Server-Design]]), so it will honour quality flags and age. Rules 2–4 stay anyway: they cost little, and third-party Modbus tools will also read the map.

## Prior art doing exactly this

- **LoRaBus** (sfera-labs): an RS-485 gateway exposing remote LoRa nodes as Modbus registers. Same topology as ours.
- **Dragino MS48-LR**: writes LoRaWAN uplink data into Modbus FC3 registers; cache-as-register-image, no synchronous proxying.
- **Sparkplug B**: birth/death certificates are the modern equivalent of the quality flag.

## Related

- [[Asynchronous-Command-Job-Pattern]]
- [[Modbus-Register-Map-Design]]
- [[Master-LoRa-Gateway-Role]]

## Sources

- `Specs/specs/01-modbus-register-map.md` §2, §6, §9 (v0.1)
- Research synthesis, 2026-09-09. https://flowfuse.com/blog/2026/04/modbus-polling-best-practices/ · https://github.com/sfera-labs/lorabus-arduino

## History

- 2026-10-14: noted that the server is now in-house; rules unchanged.
- 2026-10-21: aligned with spec pack v0.1 (50 ms response, two ages per device block, 2-bit comm state, 360 s stale threshold).

Back to [[Index]]
