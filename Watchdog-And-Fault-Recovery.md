# Watchdog And Fault Recovery

## IWDG vs WWDG

| | IWDG | WWDG |
|---|---|---|
| Clock | **LSI (~32 kHz), independent of the main clock** | APB — dies with the main clock |
| Once enabled | **cannot be disabled except by reset** | can be |
| Refresh | anytime | **only within a window** — too early *or* too late resets |
| Early warning | no | **yes — interrupt before reset** |

## The scheme

1. **IWDG, ~4 s, enabled by option byte (`IWDG_SW=0`)** so it is armed by hardware *before your code runs*. Then a bricked-at-startup image still recovers. **This is the difference between a field visit and no field visit.**
2. **Refresh only from `t_hk` (lowest priority)**, and only after checking a **per-task heartbeat array**: each task writes `hb[i] = tick` in its loop; `t_hk` verifies every heartbeat is fresher than that task's individual deadline. A stuck high-priority task starves `t_hk` → reset. A stuck low-priority task trips its own deadline.

   > **Never refresh the IWDG from a timer ISR.** That defeats the entire mechanism, and it is the most common watchdog bug in shipped firmware.

3. **WWDG additionally**, using its early-warning interrupt to **dump crash context** (task name, PC, LR, fault registers) into RTC backup registers before the reset lands. This is how you debug a field failure you cannot reproduce.
4. ⚠ LSI accuracy is loose (typically ±5% over temperature). Size the timeout with ≥30% margin, and calibrate LSI against HSE via timer input capture at boot if you need tight numbers.

## Brownout and power supervision

- **Set the BOR option byte to BOR1/BOR2 (~2.0–2.2 V)**, not the lowest level. The default lets the MCU run into the region where **flash writes are unreliable**.
- **Enable the PVD** at a threshold *above* BOR (~2.5 V) and take its interrupt: stop all flash/NOR writes, flush the FRAM counters, set a "dirty shutdown" flag, spin. This converts a possible corrupted-flash event into a clean one.
- A 470 µF bulk cap on 3.3 V buys ~10 ms at 20 mA — **enough for a FRAM write, not for a flash page erase. Hence FRAM for the hot counters** ([[Persistence-And-Storage]]).
- Monitor VDD against `VREFINT` and expose it. **Slow VDD sag is how a dying AC-DC module announces itself weeks in advance.**

## Fault handlers

A `while(1)` in `HardFault_Handler` is a bug.

```
HardFault (naked asm) → capture stacked R0-R3, R12, LR, PC, PSR + MSP/PSP
                      → read CFSR, HFSR, DFSR, MMFAR, BFAR
                      → pack into RTC backup registers (magic + regs)
                      → NVIC_SystemReset()
```

On the next boot, startup checks the magic, moves the record into the log and into a **Modbus-readable "last crash" register block**, increments `crash_count`, clears it. Now a technician reading Modbus can tell you the PC of the fault.

Also **enable the MPU** with a guard region below each task stack and at address 0. `configCHECK_FOR_STACK_OVERFLOW = 2` catches overflow after the fact; an MPU guard page catches it at the instruction.

## Error counters

All exposed as input registers — 2 bytes each, and they are the entire basis of remote diagnosis:

`modbus_frames_rx, modbus_crc_err, modbus_addr_mismatch, modbus_exceptions_sent, uart_framing_err, uart_parity_err, rs485_echo_mismatch, radio_tx_ok, radio_tx_timeout, radio_rx_ok, radio_rx_crc_err, radio_busy_timeout, radio_resets, device_timeouts[3], flash_write_err, nor_err, fram_err, crash_count, iwdg_reset_count, bor_reset_count, cfg_load_fail`

See [[Gateway-Observability]] for the full diagnostic surface.

## Escalation ladder

Each step logged, each count persisted in FRAM:

| Trigger | Action |
|---|---|
| Radio op timeout | Re-issue once |
| 3 consecutive radio timeouts | `NRESET` + full radio re-init |
| 10 consecutive failures | Re-init + drop to the most robust legal PHY config |
| Radio dead >30 min | Set `fault_bitmap.RADIO`; keep serving Modbus with stale-flagged cache; **continue the autonomous lighting schedule** |
| No valid Modbus frame >30 min (configurable) | Set `fault_bitmap.SERVER_LOST`; switch to autonomous mode |
| Storage write failures >5 | Mark storage read-only, keep running |
| `crash_count` >5 within 10 min of boot | Bootloader reverts to the previous image |

Note the posture throughout: **degrade a function, never stop lighting the road.** See [[Fail-Safe-Lighting-Rule]] and [[Failure-Modes-And-Mitigations]].

## Related

- [[Firmware-Architecture]]
- [[Firmware-Update-Over-LoRa]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://controllerstech.com/iwdg-and-wwdg-in-stm32/ · https://stm32world.com/wiki/STM32_Watchdogs · https://blog.carrese.eu/articles/2026-06-15-stm32-iwdg-watchdog-production/
