# Firmware Architecture

How the gateway firmware is structured: one FreeRTOS codebase for both Master and Slave gateways, with the role chosen by configuration. The bottom line is five statically allocated tasks, one task that owns every radio in the box, and a strict layering that keeps the protocol code host-testable.

## Key points

- **FreeRTOS 11.x via CMSIS-RTOS2**, five tasks, all statically allocated, **no `malloc` after init**.
- **Platform:** general-purpose STM32 (NUCLEO-L476RG recommended: Cortex-M4F, 80 MHz, 1 MB dual-bank flash, 128 KB RAM, RNG, no AES) with **external SX1276 modules**. See [[STM32-Part-Selection]].
- **Same codebase, role by config.** Master = 2 radios + RS485 Modbus RTU slave. Slave = 4 radios on two SPI buses (each radio of a pair on a different bus), no Modbus link to the server. FRAM and NOR sit on a third SPI bus.
- **`t_radio` owns all radios in the box** and enforces in-box TX blanking, because one radio transmitting deafens the others ([[Dual-Radio-Redundancy-And-Coexistence]]).
- **RS485 receive uses circular DMA + IDLE**, with a software t3.5 timer deciding frame boundaries.
- **Nothing in `app/` includes `stm32l4xx_hal.h`.** That's what makes the MAC and Modbus code testable on a PC.

## Why an RTOS, not a superloop

Three independent activities with very different timing:

1. **RS485 Modbus slave** (master only): must detect the Modbus frame gaps (t1.5 = 0.86 ms, t3.5 = 2.0 ms at the default 19200 8E1; fixed 0.75 / 1.75 ms above 19200) and start its response within 50 ms. Short, bounded, latency-sensitive.
2. **LoRa scheduling:** beacons, slots, retries across 2–4 radios, on a 120 s cycle with 300 ms cell slots and 2 s tier-1 slots. Long waits, ±20 ms slot accuracy.
3. **Cache and housekeeping:** persistence, watchdog, RTC, logs. Soft deadlines.

A superloop forces every radio operation into a hand-written state machine, and the Modbus latency pushes framing into ISRs anyway. **You end up building a scheduler**, so use a real one.

FreeRTOS is the default: small (~6–10 KB RAM for five tasks), fast (~223 cycles per context switch on a Cortex-M4 vs ~524 for Zephyr), and in every ST example. **Zephyr** would only be worth it with one build system across gateway and device firmware; the devices are on Blue Pills owned by another developer, so it isn't. **ThreadX** only matters for IEC 61508-style certification, which street lighting doesn't need.

## Task decomposition

| Task | Prio | Stack | Role |
|---|---|---|---|
| `t_modbus` | **5 (highest)** | 1024 W | Master only. Blocks on `q_rs485_frame`. Validates CRC/address, dispatches the function code, reads the cache under a short mutex, kicks TX DMA. **Never blocks on anything but its own queue.** |
| `t_radio` | 4 | 1024 W | **Owns every radio in the box exclusively.** Blocks on `q_radio_evt` (DIO0/DIO1 EXTI + timeouts). Runs one TX/RX state machine per radio instance, and the box-level **TX blanking**: while any radio transmits, RX on the others is discarded and not counted as link loss. |
| `t_sched` | 3 | 768 W | MAC and policy: beacons, slot schedule, retries, command dispatch, failover decisions. Talks to `t_radio` only through queues. |
| `t_storage` | 2 | 1024 W | LittleFS + FRAM. **All** flash/NOR writes are serialised here and never happen inside a Modbus response path. |
| `t_hk` | 1 (lowest) | 512 W | RTC, temperature, counter roll-ups, and the **watchdog supervisor** ([[Watchdog-And-Fault-Recovery]]). |
| ISRs | — | — | RS485 RX (DMA + IDLE), TX complete, DIO0/DIO1 EXTI per radio, timers. All under 20 µs, **post to queues only.** |

Keep the framing ISRs RTOS-aware. FreeRTOS interrupt latency (~101 cycles, ~1.3 µs at 80 MHz) is far inside the 750 µs t1.5 budget, so **don't push them above `configMAX_SYSCALL_INTERRUPT_PRIORITY`**.

## The RS485 receive path, the part people get wrong

Use **`HAL_UARTEx_ReceiveToIdle_DMA()` with *circular* DMA**, taking the HT/TC/IDLE callbacks.

One-shot DMA stops on IDLE, and **bytes arriving between the callback and the re-arm are lost**. That's exactly what happens when a fast server sends back-to-back frames. Circular DMA never stops, and the hardware IDLE detector fires after one frame time of silence (~1.15 ms at 9600 baud) at **zero CPU cost**.

```
DMA circular buffer (512 B) ← USART RX, never stopped
  IDLE / HT / TC callback:
      compute new bytes from the NDTR delta
      copy into the frame assembly buffer
      restart the t3.5 one-shot timer
  t3.5 timer ISR:
      frame boundary → xQueueSendFromISR(q_rs485_frame, &frame)
```

Use both: IDLE to wake cheaply, the explicit t3.5 timer for the spec-correct boundary. TX via `HAL_UART_Transmit_DMA()` with hardware Driver Enable (`DEM=1`); take `TC`, not `TXE`. The hardware DE timing can't enforce the 3.5-character gap, so that stays in software ([[RS485-Hardware-Design]]).

## Queues and buffers

- **Ring buffers** for byte streams: single producer / single consumer, power-of-two size, `volatile uint32_t` head/tail with masking, and `__DMB()` between writing the payload and updating the index. No lock needed on a single core when each index has one writer.
- **RTOS queues** for messages: copy-by-value for small structs (≤32 B), or pointers into a **static pool**.
- **No `malloc` after init:** `configSUPPORT_DYNAMIC_ALLOCATION=0`, everything created statically. This removes a whole class of long-uptime failures.
- The **telemetry cache** is a static array behind a **mutex held for microseconds**: copy the block out, release, *then* format the response. **Never hold a mutex across a UART transmit or a radio operation.** Sized for all devices on the site (up to 60 per Slave, count TBC); at ~32 B per device this is a few KB to ~16 KB, comfortable in 128 KB RAM.
- Give every queue a **high-water-mark counter** in the diagnostic registers. When the field misbehaves at 3 a.m., these numbers tell you which layer is starving.

## Non-blocking radio driver (SX1276, N instances)

The radio driver is the **LoRaMac-node `sx1276` driver behind our own `radio_if.h`**, extended to handle N instances (2 on the Master, 4 on a Slave). Each instance has its own SPI handle, NSS, DIO0, DIO1 and NRESET.

```
IDLE → TX_PENDING → TX_ACTIVE → TX_DONE
     → RX_ARMED   → RX_ACTIVE → RX_DONE
(any) → FAULT → RESET → IDLE
```

1. **Every state has a timeout.** `TX_ACTIVE` = computed airtime × 2 + 50 ms. `RX_ARMED` from the schedule slot.
2. **On timeout:** increment a counter, check the radio still answers on SPI (e.g. read the version register ⚠ confirm expected value against the SX1276 datasheet), log, then pulse `NRESET` and fully re-initialise. **A gateway that can't recover its own radio without a reboot will get power-cycled by a maintenance crew.**
3. **Continuous RX on SX1276 has a FIFO-pointer trap:** the RX address pointer is never reset in continuous mode, so the driver must read `RegFifoRxCurrentAddr` and set `RegFifoAddrPtr` for every packet. Give this its own host test ([[Continuous-Receive-Downlink]]).
4. **Compute airtime in firmware and refuse anything over the dwell budget:** one `lora_airtime_ms()` function, host-unit-tested, gates every TX. Under the adopted profile (915–918 MHz, ≤16 dBm EIRP, ≤400 ms per TX, ≤1% duty per box) that rules out SF11/SF12; v1 uses SF7 (SF8 allowed) on tier 1 and SF9 in cells, and no SF10. The duty ledger is a rolling 1-hour window kept **per box**, not per radio, with a 25% reserve only P0 may use ([[LoRa-Airtime-Budget]]).
5. **`t_radio` is the only thing that touches any radio.** That's what makes the whole MAC testable against a fake radio.

## Layering, the prerequisite for testing

```
app/     ← pure C, 100% host-testable, NO HAL includes
  modbus_codec.c  modbus_regmap.c  lora_mac.c  lora_airtime.c
  telemetry_cache.c  schedule.c  ota_verify.c
port/    ← thin, HAL-touching, mocked in tests
  radio_if  uart_if  store_if  time_if
bsp/     ← CubeMX-generated. Never hand-edited outside USER CODE blocks.
```

**Enforce in CI that nothing in `app/` includes `stm32l4xx_hal.h`.** The same `app/` protocol code, built with `mac_role = DEVICE`, becomes the device-role library for the Blue Pill devices. See [[Development-Tooling]] and [[Team-And-Ownership]].

## Related

- [[Watchdog-And-Fault-Recovery]]
- [[Persistence-And-Storage]]
- [[Modbus-Register-Map-Design]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[STM32-Part-Selection]]

## Sources

- `Specs/specs/01-modbus-register-map.md` §2, `Specs/specs/04-channel-plan-and-airtime.md`, `Specs/specs/06-gateway-hardware-requirements.md` §3 (v0.1)
- Research synthesis, 2026-09-09. RTOS comparison: https://www.siliconlogix.it/en/article/freertos-vs-zephyr-vs-threadx-choosing-the-right-rtos · circular-DMA RX pattern: https://pcbcool.com/technical-guides/stm32-variable-length-uart-rx-using-receive-to-idle-and-circular-dma/ · https://deepwiki.com/MaJerle/stm32-usart-uart-dma-rx-tx/3.2-rxtx-with-ring-buffer

## History

- 2026-09-09: written for a master-only gateway on STM32WLE5 with an on-die SX126x radio.
- 2026-10-14: rewritten for the current design: one codebase for Master and Slave, general-purpose STM32 + external SX1276 (N instances), `t_radio` owning all radios with in-box TX blanking, SX1276 FIFO trap.
- 2026-10-21: aligned with spec pack v0.1 (Modbus timing at 19200 8E1, 50 ms response, SF plan, third SPI bus for storage, cycle and slot timing).

Back to [[Index]]
