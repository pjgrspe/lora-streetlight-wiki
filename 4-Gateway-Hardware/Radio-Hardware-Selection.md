# Radio Hardware Selection

The radio is the **Semtech SX1276**, chosen by the client, used on every gateway and device as a **915 MHz module**. This note covers what that choice means for the hardware and driver, and the alternatives that were considered.

## Key points

- **SX1276 everywhere**: 2 radios on the Master, 4 on each Slave, 1 on each device. **915 MHz module variants only**: HopeRF RFM95W-915 or Ai-Thinker Ra-01H. Not Ra-01/Ra-02, which are 433 MHz.
- ⚠ **Semtech lists the SX1276 family as "not recommended for new designs."** No end-of-life date was found. Recommend a **buffer buy** for the full build plus spares, and keep the `radio_if.h` abstraction so moving to the SX1262 stays a port, not a rewrite.
- **Every radio has its own antenna connector, antenna and coax run.** Each radio of a redundant pair also has its own SPI bus, reset line and LDO/fused rail ([[Dual-Radio-Redundancy-And-Coexistence]]).
- **Radio profile (decided):** 915–918 MHz, 125 kHz channels, **≤16 dBm EIRP** (antenna gain counts, so PA output is set in firmware below that), ≤400 ms per transmission, ≤1% duty cycle per box ([[Philippines-Regulatory-Context]]).
- **Radios in one box blind each other.** In-box antenna isolation is only ~17–28 dB, far short of what simultaneous TX/RX needs. So a per-box TX scheduler is required, and ⚠ isolation must be measured on the real enclosure.

## What the SX1276 means in practice

| Aspect | SX1276 behaviour | Design consequence |
|---|---|---|
| Max TX | +20 dBm (PA_BOOST) | Irrelevant headroom: you run at or below the legal EIRP |
| Sensitivity | −136 dBm at SF12/125 kHz | SF11/12 unusable under the 400 ms dwell limit anyway ([[LoRa-Airtime-Budget]]) |
| RX current | 9.9–11.5 mA | Irrelevant on mains power ([[Continuous-Receive-Downlink]]) |
| **Continuous RX FIFO** | ⚠ The RX address pointer is never reset in continuous mode | The driver must manage `FifoAddrPtr` / `FifoRxCurrentAddr` on every packet. Known work item, needs a host test |
| **CAD** | Fixed algorithm, CadDone/CadDetected on DIO0/DIO1, no `CAD_RX` exit | Still usable as a guard before TX ([[Media-Access-Control-Options]]) |
| Regulator / XTAL | LDO only; external crystal matching | No internal TCXO control (see below) |

**Driver:** use the **Lora-net LoRaMac-node `sx1276` driver** (BSD) or Semtech's reference driver, behind your own `radio_if.h`. The driver must support **N instances** (4 on a Slave). NUCLEO-L476RG is one of LoRaMac-node's SX1276 reference boards ([[STM32-Part-Selection]]).

⚠ **Do not hand-roll the driver.** Semtech radios have documented errata, and the reference drivers already carry the workarounds.

**Architectural instruction:** write `radio_if.h` with ~10 functions (`radio_init`, `radio_set_channel`, `radio_tx`, `radio_rx_start`, `radio_get_rssi_snr`, `radio_sleep`, `radio_irq_poll`, `radio_health`) and put the vendor driver behind it. This makes an SX1276→SX1262 swap a one-file port and makes the MAC host-testable ([[Development-Tooling]]).

## Interfacing an SX1276 module

| Pin | To STM32 | Notes |
|---|---|---|
| `NSS` | GPIO, software CS | One per radio |
| `SCK/MOSI/MISO` | SPI1 or SPI2 | Primary and backup radio of a pair on **different** buses, so a stuck MISO on one can't take out both |
| `DIO0` | EXTI | TxDone / RxDone / CadDone |
| `DIO1` | EXTI | RxTimeout / CadDetected |
| `NRESET` | GPIO | One per radio. Must be in the fault-recovery path |
| Antenna | u.FL / SMA → coax → bulkhead | One per radio; see [[Enclosure-Power-And-Surge]] for coax loss |

## Frequency accuracy (TCXO)

A ±10 ppm crystal is ±9 kHz at 915 MHz. LoRa tolerates roughly ±25% of bandwidth of combined TX+RX offset: at 125 kHz that is ±31 kHz, so a ±10 ppm pair (±18 kHz worst case) *just* passes. At 62.5 kHz or below it fails. Semtech's guidance is about ±30 ppm total under all conditions.

**Verdict:** prefer **gateway modules with a TCXO option**. A gateway's frequency error is inherited by every link, and a cabinet interior reaches 60–80 °C. Otherwise accept a good crystal and keep margin by staying at **SF ≤ 9 / 125 kHz**. A good ±10 ppm crystal is acceptable on devices at SF ≤ 10.

## Alternatives considered

| Option | Status |
|---|---|
| **SX1262 / SX1268** (2nd gen) | Technically better: +22 dBm, 4.6 mA RX, parameterised CAD with `CAD_RX` exit, clean buffer model, internal XTAL trim, TCXO via DIO3. Was the original recommendation. **The future migration path** if SX1276 supply tightens |
| **STM32WLE5 / RAK3172-T** (SX126x on the MCU die, pre-certified module) | Was the original gateway recommendation for its NTC type-approval head start. Ruled out: one radio only, and not SX1276 ([[STM32-Part-Selection]]) |
| LR1121 | Over-specified (2.4 GHz, satellite); cost and ecosystem maturity |
| AT-command modules (e.g. Ebyte E22-900T22S) | Avoid: the AT layer hides CAD, precise TX timing and slot-accurate scheduling, which a custom MAC needs. Use SPI-transparent modules only |

## Type approval

Philippine type approval is mandatory per RF module. NTC accepts host approval based on the integrated module, so **prefer modules that already hold radio test reports**. ⚠ Whether NTC accepts a module's existing FCC/CE certification is unverified; that answer is worth months of schedule ([[Philippines-Regulatory-Context]], [[Open-Questions]]).

## Related

- [[STM32-Part-Selection]]
- [[Dual-Radio-Redundancy-And-Coexistence]]
- [[Continuous-Receive-Downlink]]
- [[Philippines-Regulatory-Context]]

## Sources

- Research synthesis, 2026-09-09. SX1261/2 datasheet Rev 1.2: https://cdn.sparkfun.com/assets/6/b/5/1/4/SX1262_datasheet.pdf · https://github.com/Lora-net/sx126x_driver · https://docs.rakwireless.com/product-categories/wisduo/rak3172-module/overview/
- Semtech SX1276 "not recommended for new design" status: https://www.semtech.com/products/wireless-rf/lora-connect/sx1276dvk1ias · https://forum.lora-developers.semtech.com/t/sx1276-nrfnd/1265
- Project owner, 2026-10-14: client specified SX1276 on STM32 boards.

## History

- 2026-09-09: SX1262 (inside STM32WLE5 / RAK3172-T) recommended; SX1276 "acceptable only with inventory".
- 2026-10-14: Client specified SX1276. Note rewritten around it; SX1262 kept as the migration path.

Back to [[Index]]
