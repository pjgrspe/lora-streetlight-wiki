# Gateway Observability

What the gateways expose so a fault can be diagnosed remotely or by a technician on site. The bottom line: every counter is readable over Modbus (via the Master) and on a serial console, with simple LEDs for a no-laptop first look. This is a P1 feature, not a later addition.

## Key points

- **Governing rule:** a diagnostic that exists only on a debug UART inside a sealed enclosure, in the rain, doesn't exist. Every counter must reach Modbus (Master, plus Slave summaries relayed upward) **and** the console.
- **Radio diagnostics are per radio**, and include which radio of each pair is active and the failover count ([[Dual-Radio-Redundancy-And-Coexistence]]).
- **Judge a link by RSSI when SNR ≥ 7 dB, by SNR below that.**
- **Budget ~25 dB of fade margin** in tropical, vegetated conditions; foliage loss is seasonal.
- **The serial console is the cheapest high-value diagnostic you'll build.**

## What to expose

| Group | Points |
|---|---|
| Per radio | `radio_index, role (primary/backup/up/down), active, channel, sf_current, tx_power, noise_floor, tx/rx counts, crc_err, resets, failovers, blanked_rx_count` |
| Per link | `rssi_last, rssi_avg, snr_last, snr_min`. **Both directions: asymmetry is diagnostic gold** |
| Per node reliability | `tx_count, rx_count, ack_count, retry_count, timeout_count, per_pct (rolling 1 h / 24 h), last_seen_utc, consecutive_misses` |
| Per node lifecycle | `uptime, reboot_count, last_reboot_reason, fw_version, config_version_hash`. Reboot-reason codes catch brownouts and watchdog loops |
| Per device errors | `error_code, error_count, first/last occurrence`, a **documented enum**, not ad-hoc bitfields |
| Device-reported DALI status | `level_actual, lamp_fail, gear_fail, fault_code`, as reported by each per-pole device (the devices own the DALI link) |
| Gateway internals | `queue_depths (P0..P3), duty_cycle_used_pct (per box), buffer_fill_pct, dropped_events, stack watermarks, cpu_load, temperature, rs485_frame_errors, modbus_exception_count` |
| Time | `utc_now, last_sync_utc, sync_source, estimated_drift_ms, clock_trusted` |

## Reading signal quality in the field

Worth a laminated card for the install crew:

- LoRa SNR typically ranges −20 to +10 dB; LoRa demodulates 7.5–20 dB *below* the noise floor depending on SF.
- **If SNR ≥ 7 dB, judge the link by RSSI. If SNR < 7 dB, judge it by SNR.**
- As SNR approaches the per-SF limit (about −10 dB at SF8), reception starts failing.
- Budget roughly **25 dB of margin** in tropical, vegetated conditions. A link commissioned in the dry season with 8 dB of margin will fail in the wet season.
- On a dual-radio box, **compare the two radios of a pair.** If one is consistently 6+ dB worse, suspect its antenna, coax or connector (water ingress) before the radio.

## Three surfaces

**1. LEDs, no laptop needed.** Minimum four, meanings printed on the enclosure label. The Nucleo dev boards have one user LED, so this belongs in the gateway hardware requirements for the final board:

| LED | Meaning |
|---|---|
| `PWR` | Solid |
| `RS485` | Blink per valid Modbus frame (Master) |
| `LORA` | Blink per TX, double-blink per RX, slow blink = running on backup radio, fast = no peers |
| `FAULT` | Off = OK, slow = warning, fast = critical |

**2. Serial console, the workhorse.** On the dev boards it's the ST-LINK virtual COM port. A line-oriented CLI:

```
status              nodes            link <addr>       ping <addr>
radios              failover <pair>  job <id>          log tail
rf scan             set-time         commission        factory-reset
```

`nodes` prints a table: address, type, last seen, RSSI, SNR, PER, firmware. `radios` shows each radio's role, channel, health and which one is active. Human-readable **and** a `--json` mode for scripts. Plus a rolling log (~2000 events with reason codes) that survives reboots.

**3. Optional local web or BLE service port.** Useful for pole-top work but adds attack surface. Gate it behind a physical button with a short session timeout. Not P1.

## Related

- [[Failure-Modes-And-Mitigations]]
- [[Modbus-Register-Map-Design]]
- [[Watchdog-And-Fault-Recovery]]
- [[Dual-Radio-Redundancy-And-Coexistence]]

## Sources

- Research synthesis, 2026-09-09. RSSI vs SNR judgement rule: https://www.thethingsnetwork.org/docs/lorawan/rssi-and-snr/ · tropical/forest path loss at 923 MHz: https://pmc.ncbi.nlm.nih.gov/articles/PMC13210814/

## History

- 2026-09-09: written for a single-radio master with DALI diagnostics gathered through the gateway.
- 2026-10-14: diagnostics made per radio with failover state. DALI status is now reported by per-pole devices. Applies to Master and Slave gateways.

Back to [[Index]]
