# Firmware Architecture

## Superloop or RTOS?

Three mutually independent activities with different timing characters:

1. **RS-485 Modbus slave** — must detect a 1.72 ms intra-frame gap and respond inside the master's timeout. Latency-sensitive, short, bounded.
2. **LoRa scheduler** — polls 3 slaves, each round trip 100 ms–2 s. Slow, with long blocking waits.
3. **Cache and housekeeping** — persistence, watchdog, RTC, logs. Soft.

A superloop is workable, but activity 2's long waits force you to hand-decompose every radio operation into a state machine with explicit timeouts, and activity 1's latency pushes framing into ISRs anyway. **You end up building a scheduler.**

**Recommendation: FreeRTOS 11.x via CMSIS-RTOS2**, configured conservatively. ~223 cycles per context switch and ~101 cycles of interrupt latency on a Cortex-M4 (versus Zephyr's ~524/~143), ~6–10 KB RAM for five tasks, and it is in every ST example you will copy.

**Reconsider Zephyr only if** you also own the slave, meter and node firmware and want one build system across all four products — its `lora` and `modbus` subsystems plus MCUboot are then genuinely compelling, but you'd want the G0B1/L4 flash headroom. **ThreadX** only if you need IEC 61508 / ISO 26262 certification evidence, which street lighting does not.

## Task decomposition

| Task | Prio | Stack | Role |
|---|---|---|---|
| `t_modbus` | **5 (highest)** | 1024 W | Blocks on `q_rs485_frame`. Validates CRC/addr, dispatches FC, reads the cache under a short mutex, kicks TX DMA. **Never blocks on anything but its own queue.** |
| `t_radio` | 4 | 1024 W | Owns the radio *exclusively*. Blocks on `q_radio_evt` (DIO1 EXTI + timeouts). Runs the TX/RX state machine. |
| `t_sched` | 3 | 768 W | The MAC/policy layer: poll requests, beacons, retries, command dispatch. Talks to `t_radio` only through queues. |
| `t_storage` | 2 | 1024 W | LittleFS + FRAM. **All** flash/NOR writes serialised here — flash ops are slow and must never happen inside a Modbus response path. |
| `t_hk` | 1 (lowest) | 512 W | RTC, temperature, counter rollups, and the **watchdog supervisor**. |
| ISRs | — | — | RS485 RX (DMA+IDLE), TX complete, DIO1 EXTI, timers. All <20 µs, **post to queues only.** |

Keep the framing ISRs RTOS-aware; FreeRTOS's ~101-cycle latency at 48 MHz is 2.1 µs — three orders of magnitude inside the 750 µs t1.5 budget. **Don't over-engineer this** by pushing them above `configMAX_SYSCALL_INTERRUPT_PRIORITY`.

## The RS-485 receive path — the part people get wrong

Use **`HAL_UARTEx_ReceiveToIdle_DMA()` with *circular* DMA**, taking HT/TC/IDLE callbacks.

Why circular and not one-shot: one-shot stops the DMA on IDLE, and between the IDLE callback and your re-arm, **bytes are lost** — exactly what happens when a fast master sends back-to-back frames. Circular DMA never stops, and the hardware IDLE detector fires after one full frame time of silence (~1.15 ms at 9600), conveniently just under t1.5, at **zero CPU cost**.

```
DMA circular buffer (512 B) ← USART RX, never stopped
  IDLE / HT / TC callback:
      compute new bytes from the NDTR delta
      copy into the frame assembly buffer
      restart the t3.5 one-shot timer
  t3.5 timer ISR:
      frame boundary → xQueueSendFromISR(q_rs485_frame, &frame)
```

**Use both:** IDLE to wake cheaply, the explicit t3.5 timer to decide the spec-correct boundary. TX via `HAL_UART_Transmit_DMA()` with `DEM=1`; take `TC`, not `TXE`.

## Queues and buffers

- **Ring buffers** for byte streams: single-producer/single-consumer, power-of-two size, `volatile uint32_t` head/tail with masking rather than modulo, and `__DMB()` between the payload write and the index update. No locking needed on a single-core M4 if only one side writes each index.
- **RTOS queues** for messages: copy-by-value for small structs (≤32 B), or pointers into a **static pool**.
- **No `malloc` after init.** Set `configSUPPORT_DYNAMIC_ALLOCATION=0` and create everything statically. This eliminates an entire class of long-uptime failure.
- The **telemetry cache** is a plain static array protected by a **mutex held for microseconds** — memcpy the block out, release, *then* format the response. **Never hold a mutex across a UART transmit or a radio operation.**
- Give every queue a **high-water-mark counter** exposed in the diagnostic registers. When the field misbehaves at 3 a.m., these are the numbers that tell you which layer is starving.

## Non-blocking radio driver

```
IDLE → TX_PENDING → TX_ACTIVE → TX_DONE
     → RX_ARMED   → RX_ACTIVE → RX_DONE
(any) → FAULT → RESET → IDLE
```

1. **Every state has a timeout.** `TX_ACTIVE` = computed airtime × 2 + 50 ms. `RX_ARMED` from the schedule slot. BUSY-wait = 1 ms (10 ms after calibration).
2. On timeout: increment a counter, call `GetDeviceErrors()`, log, then `NRESET` + full re-init. **A gateway that cannot recover its own radio without a reboot is a gateway that will be power-cycled by a maintenance crew.**
3. **Compute airtime in firmware and refuse to transmit anything over the dwell budget** — one `lora_airtime_ms()` function, host-unit-tested, guarding every TX. See [[LoRa-Airtime-Budget]].
4. `t_radio` is the *only* thing that touches the radio. That is what makes the whole MAC testable against a fake radio.

## Layering — the prerequisite for testing

```
app/     ← pure C, 100% host-testable, NO HAL includes
  modbus_codec.c  modbus_regmap.c  lora_mac.c  lora_airtime.c
  telemetry_cache.c  schedule.c  ota_verify.c
port/    ← thin, dumb, HAL-touching. Mocked in tests.
  radio_if  uart_if  store_if  time_if
bsp/     ← CubeMX-generated. Never hand-edited outside USER CODE blocks.
```

**Enforce that nothing in `app/` includes `stm32wlxx_hal.h`.** See [[Development-Tooling]].

## Related

- [[Watchdog-And-Fault-Recovery]]
- [[Persistence-And-Storage]]
- [[Modbus-Register-Map-Design]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. RTOS comparison: https://www.siliconlogix.it/en/article/freertos-vs-zephyr-vs-threadx-choosing-the-right-rtos · circular-DMA RX pattern: https://pcbcool.com/technical-guides/stm32-variable-length-uart-rx-using-receive-to-idle-and-circular-dma/ · https://deepwiki.com/MaJerle/stm32-usart-uart-dma-rx-tx/3.2-rxtx-with-ring-buffer
