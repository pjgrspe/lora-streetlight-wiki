# Gateway Observability

The difference between a two-hour and a two-day site visit. Diagnostics are a P1 feature of the master gateway, not a later addition.

**Governing rule: every counter must be readable from all three surfaces below, and over Modbus.** A diagnostic that exists only on a debug UART inside a sealed enclosure, in the rain, does not exist.

## What to expose

| Group | Points |
|---|---|
| Per-link radio | `rssi_last, rssi_avg, snr_last, snr_min, sf_current, tx_power, freq_channel, noise_floor` — **both directions; asymmetry is diagnostic gold** |
| Per-node reliability | `tx_count, rx_count, ack_count, retry_count, timeout_count, crc_err_count, per_pct (rolling 1 h / 24 h), last_seen_utc, consecutive_misses` |
| Per-node lifecycle | `uptime, reboot_count, last_reboot_reason, fw_version, config_version_hash, vbat` — reboot-reason codes catch brownouts and watchdog loops |
| Per-device errors | `error_code, error_count, first/last occurrence` — a **documented enum**, not ad-hoc bitfields |
| DALI subsystem | `bus_voltage, bus_fault, addresses_discovered, addresses_responding, per_address_fault_code, lamp_fail_count, gear_fail_count` |
| Gateway internals | `queue_depths (P0..P3), duty_cycle_used_pct, buffer_fill_pct, dropped_events, heap/stack watermark, cpu_load, temperature, rs485_frame_errors, modbus_exception_count` |
| Time | `utc_now, last_sync_utc, sync_source, estimated_drift_ms, clock_trusted` |

## Reading signal quality in the field

Worth printing on a laminated card for the install crew:

- LoRa SNR typically ranges −20 to +10 dB; LoRa demodulates 7.5–20 dB *below* the noise floor depending on SF.
- **If SNR ≥ 7 dB, judge the link by RSSI. If SNR < 7 dB, judge it by SNR.**
- As SNR approaches the per-SF limit (about −10 dB at SF8), reception begins to fail.
- Budget roughly **25 dB of margin** in tropical, vegetated conditions. Foliage attenuation is **seasonal** — a link commissioned in the dry season with 8 dB of margin will fail in the wet season.

## Three surfaces

**1. LEDs — no laptop required.** Minimum four, with semantics printed on the enclosure label:

| LED | Meaning |
|---|---|
| `PWR` | Solid |
| `RS485` | Blink per valid Modbus frame |
| `LORA` | Blink per TX, double-blink per RX, slow blink = degraded, fast = no peers |
| `FAULT` | Off = OK, slow = warning, fast = critical |

**2. Serial/USB console — the workhorse, and the cheapest high-value diagnostic you will build.** A line-oriented CLI:

```
status              nodes            link <addr>       ping <addr>
dali scan           job <id>         log tail          rf scan
set-time            commission       factory-reset
```

`nodes` prints a table: addr, type, last-seen, RSSI, SNR, PER, fw. Human-readable **and** a `--json` mode so scripts can scrape it. Plus a rolling in-flash log (~2000 events, timestamped, with reason codes) that survives reboots.

**3. Optional local web or BLE service port.** Genuinely useful for pole-top work, but it adds attack surface — gate it behind a physical button press with a short session timeout. Not P1.

## Related

- [[Failure-Modes-And-Mitigations]]
- [[Modbus-Register-Map-Design]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. RSSI vs SNR judgement rule: https://www.thethingsnetwork.org/docs/lorawan/rssi-and-snr/ · tropical/forest path loss at 923 MHz: https://pmc.ncbi.nlm.nih.gov/articles/PMC13210814/
