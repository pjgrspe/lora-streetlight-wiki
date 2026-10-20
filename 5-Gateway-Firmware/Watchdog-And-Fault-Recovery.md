# Watchdog And Fault Recovery

How the gateway detects that it's stuck or dying, records why, and recovers without a site visit. The bottom line: a hardware-armed independent watchdog fed only by the lowest-priority task after checking every task's heartbeat, crash context captured for remote reading, and an escalation ladder that degrades functions but never stops the lighting schedule.

## Key points

- **IWDG armed by option byte**, so it runs before any code does, and a bricked-at-startup image still recovers.
- **Refresh only from `t_hk`, and only after every task's heartbeat checks out.** Never from a timer ISR.
- **Capture crash context** (PC, LR, fault registers) into RTC backup registers, and expose it over Modbus and the console.
- **Set BOR above the default and use the PVD interrupt** to stop flash writes cleanly on a brownout.
- **Escalate per radio:** a failing radio is reset, then failed over to its pair ([[Dual-Radio-Redundancy-And-Coexistence]]), and the lighting schedule never stops.

## IWDG vs WWDG

| | IWDG | WWDG |
|---|---|---|
| Clock | **LSI (~32 kHz), independent of the main clock** | APB, dies with the main clock |
| Once enabled | **can't be disabled except by reset** | can be |
| Refresh | anytime | **only inside a window**; too early *or* too late resets |
| Early warning | no | **yes, an interrupt before reset** |

## The scheme

1. **IWDG, ~4 s, enabled by option byte (`IWDG_SW = 0`)**, armed by hardware *before your code runs*. **This is the difference between a field visit and no field visit.**
2. **Refresh only from `t_hk` (lowest priority)**, after checking a **per-task heartbeat array**: each task writes `hb[i] = tick` in its loop, and `t_hk` checks each heartbeat against that task's own deadline. A stuck high-priority task starves `t_hk`, so the IWDG fires. A stuck low-priority task misses its own deadline.

   > **Never refresh the IWDG from a timer ISR.** That defeats the whole mechanism, and it's the most common watchdog bug in shipped firmware.

3. **WWDG as well**, using its early-warning interrupt to **dump crash context** (task name, PC, LR, fault registers) into RTC backup registers before the reset lands. This is how you debug a field failure you can't reproduce. ⚠ Confirm the backup-register count on the L476 (expected 32).
4. ⚠ LSI accuracy is loose (around ±5% over temperature). Size the timeout with ≥30% margin, or calibrate LSI against the main clock at boot.

## Brownout and power supervision

- **Set the BOR option byte to a mid level (~2.0–2.2 V)**, not the lowest. The default lets the MCU run into the region where **flash writes are unreliable**.
- **Enable the PVD** at a threshold *above* BOR (~2.5 V) and take its interrupt: stop all flash/NOR writes, flush FRAM counters, set a "dirty shutdown" flag, then wait. A possible corrupted-flash event becomes a clean one.
- A 470 µF bulk cap on 3.3 V buys ~10 ms at 20 mA. That's **enough for a FRAM write, not for a flash page erase**, hence FRAM for the hot counters ([[Persistence-And-Storage]]). With 2–4 radios on board, budget the hold-up current for the whole box, not just the MCU.
- Monitor VDD against `VREFINT` and expose it. **Slow VDD sag is how a dying AC-DC module announces itself weeks ahead.**

## Fault handlers

A `while(1)` in `HardFault_Handler` is a bug.

```
HardFault (naked asm) → capture stacked R0-R3, R12, LR, PC, PSR + MSP/PSP
                      → read CFSR, HFSR, DFSR, MMFAR, BFAR
                      → pack into RTC backup registers (magic + regs)
                      → NVIC_SystemReset()
```

On the next boot, startup checks the magic, moves the record into the log and into the log and the console (any gateway), increments `crash_count`, and clears it. Over Modbus, the v1 map exposes `reset_cause` (including "fault handler" and "watchdog"), `boot_count`, fault bit 8 (last reset was a watchdog reset) and the `MASTER_BOOT` event (`Specs/specs/01-modbus-register-map.md` §6). A full "last crash" register block is a candidate for a later minor version of the map, added in reserved space. A technician can then read the PC of the fault without a debugger.

Also **enable the MPU** with a guard region below each task stack and at address 0. `configCHECK_FOR_STACK_OVERFLOW = 2` catches overflow after the fact; an MPU guard catches it at the faulting instruction.

## Error counters

Exposed as registers on the Master, and on every gateway through the console. They're the basis of remote diagnosis:

`modbus_frames_rx, modbus_crc_err, modbus_addr_mismatch, modbus_exceptions_sent, uart_framing_err, rs485_echo_mismatch,` **per radio:** `radio_tx_ok, radio_tx_timeout, radio_rx_ok, radio_rx_crc_err, radio_resets, radio_failovers,` **per device:** `timeouts,` and `flash_write_err, nor_err, fram_err, crash_count, iwdg_reset_count, bor_reset_count, cfg_load_fail`.

See [[Gateway-Observability]] for the full diagnostic surface.

## Escalation ladder

Each step logged, each count persisted in FRAM:

| Trigger | Action |
|---|---|
| Radio operation timeout | Re-issue once |
| 3 consecutive timeouts on one radio | `NRESET` + full re-init of that radio |
| 10 consecutive failures on one radio | Mark it failed; **fail over to its pair** and raise an alarm ([[Dual-Radio-Redundancy-And-Coexistence]]) |
| Both radios of a pair dead > 30 min | Set the radio fault bits (`fault_bitmap` bits 1–2 on the Master); keep serving Modbus with stale-flagged cache; **continue the autonomous lighting schedule** |
| No valid Modbus frame > 30 min (Master, configurable) | Raise `SERVER_LOST` (event `0x8013`, read by the server on reconnect); run schedules autonomously; keep buffering events |
| Storage write failures > 5 | Mark storage read-only, keep running |
| `crash_count` > 5 within 10 min of boot | Revert to the previous firmware image ([[Firmware-Update-Over-LoRa]]) |

The posture throughout: **degrade a function, never stop lighting the road.** See [[Fail-Safe-Lighting-Rule]] and [[Failure-Modes-And-Mitigations]].

## Related

- [[Firmware-Architecture]]
- [[Firmware-Update-Over-LoRa]]
- [[Gateway-Observability]]
- [[Persistence-And-Storage]]

## Sources

- `Specs/specs/01-modbus-register-map.md` §6, §7.3; `Specs/specs/05-fail-safe-behaviour.md` F1, F17 (v0.1)
- Research synthesis, 2026-09-09. https://controllerstech.com/iwdg-and-wwdg-in-stm32/ · https://stm32world.com/wiki/STM32_Watchdogs · https://blog.carrese.eu/articles/2026-06-15-stm32-iwdg-watchdog-production/

## History

- 2026-09-09: written for a single-radio master gateway.
- 2026-10-14: counters and escalation made per radio, with failover to the paired radio before declaring the radio link dead. Applies to Master and Slave gateways.
- 2026-10-21: aligned with spec pack v0.1 (what the v1 Modbus map exposes about resets; SERVER_LOST as an event; fault bits).

Back to [[Index]]
