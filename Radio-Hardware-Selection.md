# Radio Hardware Selection

> **Superseded 2026-10-14: the client has specified the SX1276**, on STM32 boards, throughout (14 gateway radios plus every device). The SX1262/STM32WLE5 recommendation below is kept for the record. What the SX1276 decision means:
> - ⚠ **Semtech lists the SX1276 family as "not recommended for new designs"** ([Semtech](https://www.semtech.com/products/wireless-rf/lora-connect/sx1276dvk1ias), [forum](https://forum.lora-developers.semtech.com/t/sx1276-nrfnd/1265)). No end-of-life date was found. It is still widely available as modules (e.g. HopeRF RFM95W; Ai-Thinker Ra-01/Ra-02 are 433 MHz only, so **use 915 MHz variants**). Tell the client in writing, and recommend a **buffer buy** of radios for the full build plus spares. A later move to SX1262 is a one-file port **if** the `radio_if.h` abstraction below is respected.
> - **Continuous RX has the FIFO-pointer trap** ([[Continuous-Receive-Downlink]]). The driver must manage `FifoAddrPtr` / `FifoRxCurrentAddr` on every packet. Budget time for it.
> - **CAD is the fixed-algorithm kind** (CadDone/CadDetected on DIO0/DIO1, no `CAD_RX` exit). Still usable as a guard.
> - **+20 dBm PA_BOOST**, but you'll run at or below the legal EIRP anyway ([[Philippines-Regulatory-Context]]).
> - **No internal TCXO control.** Gateways should use modules with a TCXO option, or accept a crystal with frequency-offset margin at SF ≤ 9 / 125 kHz.
> - Driver: the **Lora-net LoRaMac-node `sx1276` driver** (BSD) or Semtech's reference driver behind `radio_if.h`. The driver must support **N instances** (4 on a slave).

## Chip comparison

| | SX1276/78 | **SX1262/68** | LR1121 |
|---|---|---|---|
| Generation | 1st (2013) | 2nd (2017) | 3rd (2023) |
| Max Tx | +20 dBm (PA_BOOST) | **+22 dBm** | +22 dBm |
| Sensitivity SF12/125 kHz | −136 dBm | **−137 dBm** | ~+4.5 dB better (vendor claim, unverified) |
| Rx current @125 kHz | 9.9–11.5 mA | **4.6 mA** | <7 mA |
| **CAD** | Fixed algorithm, two IRQs | **Parameterised**: `cadSymbolNum`, `cadDetPeak`, `cadDetMin`, **`cadExitMode=CAD_RX`**, `cadTimeout` | Parameterised + LR-FHSS |
| Regulator | LDO only | **LDO or DC-DC buck** | DC-DC |
| XTAL caps | External matching required | **Internal, software-trimmable** | Internal |
| Buffer model | Continuous-Rx FIFO pointer trap | Clean | Clean |

**Recommendation: the SX1262 family — ideally as the radio inside an STM32WLE5.** The parameterised CAD with `CAD_RX` exit matters for a CSMA-guarded custom MAC; the cleaner buffer model avoids the SX127x continuous-Rx trap (see [[Continuous-Receive-Downlink]]); +22 dBm gives headroom if regulation permits; and it is the best-supported target in every open-source stack.

**LR1121 is over-specified** — no need for 2.4 GHz or satellite, and you'd pay in cost and ecosystem maturity. **SX1276 is acceptable** only if you have inventory or an already-certified module; it has no advantage.

## TCXO — decide by SF and role

A ±10 ppm 32 MHz crystal is ±9 kHz at 915 MHz. LoRa tolerates roughly ±25% of bandwidth of combined TX+RX offset: at 125 kHz that is ±31 kHz, so a ±10 ppm pair (±18 kHz worst case) *just* passes. At 62.5 kHz or below it fails.

Semtech's own guidance: end-device reference accuracy should be about **±30 ppm under all operating conditions**, including initial error, temperature drift and ageing.

**Verdict:**
- **TCXO on the master and slave gateways.** A gateway's frequency error is inherited by every link, and a Philippine enclosure interior can hit 60–80 °C. A thermally drifting gateway is a whole-section outage; the cost delta is a few dollars.
- A good ±10 ppm crystal is acceptable on devices at SF ≤ 10.

SX1262 makes this easy: **DIO3 supplies and gates the TCXO** via `SetDio3AsTcxoCtrl(voltage, timeout)`, regulated 1.6–3.3 V. ⚠ **Bring-up trap:** set the startup timeout too short and the radio reports an `XoscStart` error in `GetDeviceErrors()` and silently refuses to transmit.

## SX126x interfacing (only needed for the external-radio fallback)

| Pin | To STM32 | Notes |
|---|---|---|
| `NSS` | GPIO, **software CS** | Hardware NSS is unreliable for holding CS across multi-byte opcode+payload |
| `SCK/MOSI/MISO` | SPI1 | ≤16 MHz spec; **run 8 MHz** for margin. Mode 0, MSB first |
| `BUSY` | GPIO input | **Mandatory.** Poll high→low before *every* transaction. Add a ~1 ms timeout (longer after calibration) and treat expiry as a radio fault. **The number-one SX126x driver bug.** |
| `DIO1` | EXTI | Primary IRQ: TxDone/RxDone/Timeout/CrcErr/CadDone |
| `DIO2` | *not connected* | Configure as **RF switch control** — zero switching latency relative to PA ramp |
| `DIO3` | *not connected* | Configure as **TCXO supply** |
| `NRESET` | GPIO, open-drain | ≥100 µs low. Must be in the fault-recovery path |

## Driver libraries

| Library | Fit |
|---|---|
| **STM32CubeWL `SubGHz_Phy`** | ★ For STM32WL. Start from the **`SubGHz_Phy_PingPong`** example — it is precisely "custom LoRa, no LoRaWAN" |
| **Lora-net `sx126x_driver`** (BSD-3) | ★ For external SX1262. Clean HAL-port model, no LoRaWAN baggage |
| Lora-net `LoRaMac-node` | Reference-grade PHY, drags the LoRaWAN stack. Worth *reading* for errata handling |
| RadioLib (MIT) | Great for bench prototyping; C++ and Arduino-centric — not what to ship in a gateway |

⚠ **Do not hand-roll the driver.** There is a documented "RxTimeout after RxDone" errata class whose workaround (writing specific registers after each RX) is already applied inside CubeWL's `radio_driver.c` and LoRaMac-node's `sx126x.c`. Rolling your own means rediscovering it in the field.

**Architectural instruction:** write your own `radio_if.h` with ~10 functions (`radio_init`, `radio_set_channel`, `radio_tx`, `radio_rx_start`, `radio_get_rssi_snr`, `radio_sleep`, `radio_irq_poll`, `radio_health`) and put CubeWL *behind* it. This is what makes the module→bare-chip swap a one-file port and makes the MAC host-testable — see [[Development-Tooling]].

## Pre-certified module vs bare chip

| | Bare chip | Pre-certified module |
|---|---|---|
| Unit cost | Lowest at volume | +$2–8 |
| RF design risk | **High** — matching, harmonics, ground plane, antenna port. Getting +22 dBm harmonics under limits is real work | Vendor's problem |
| **NTC type approval** | Full radio test campaign per host product | **Reuse the module's certificate** — NTC accepts host approval per the specific integrated RF module |
| Time to first unit | Months | Weeks |

**`RAK3172-T` for the first production run.** ⚠ Verify with NTC or a local test house whether the module's existing FCC/CE certificate is accepted — **that answer is worth several months of schedule.** Note: avoid AT-command modules like the `E22-900T22S` — the AT abstraction fights a custom MAC, costing you CAD control, precise TX timing and slot-accurate scheduling. If going Ebyte, use the SPI-transparent `E22-900M22S`.

## Related

- [[STM32-Part-Selection]]
- [[Philippines-Regulatory-Context]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. SX1261/2 datasheet Rev 1.2: https://cdn.sparkfun.com/assets/6/b/5/1/4/SX1262_datasheet.pdf · https://github.com/Lora-net/sx126x_driver · https://docs.rakwireless.com/product-categories/wisduo/rak3172-module/overview/
