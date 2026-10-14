# LoRa PHY Fundamentals

How the LoRa physical layer works: spreading factor, bandwidth, coding rate, and the header/preamble/CRC settings that trip up custom protocols. Bottom line: under the Philippine profile this project uses SF7–SF10 at 125 kHz and CR 4/5, and the PHY CRC is not a substitute for authentication.

## Key points

- LoRa is Chirp Spread Spectrum and demodulates **below the noise floor**. Each +1 SF buys ~2.5 dB and costs ~2× airtime.
- All radios are **SX1276**. At 125 kHz its sensitivity runs from −123 dBm (SF7) to −132 dBm (SF10), the highest SF allowed here.
- **SF11/SF12 are unusable** under the 400 ms dwell limit ([[LoRa-Airtime-Budget]]).
- Defaults: **125 kHz, CR 4/5, explicit header, PHY CRC on**.
- The preamble length must match on TX and RX, or packets drop silently.
- SFs are only quasi-orthogonal: a same-SF interferer 6 dB stronger kills your packet.

## Spreading factor

LoRa sends each symbol as a linear frequency sweep across the whole channel, `2^SF` chips per symbol. Because the chirp uses the full bandwidth, it resists narrowband interference, multipath and Doppler.

| SF | Chips/symbol | Required demod SNR | Sensitivity @125 kHz (SX1276) | Usable here? |
|---|---|---|---|---|
| 7 | 128 | −7.5 dB | −123 dBm | ✅ |
| 8 | 256 | −10 dB | −126 dBm | ✅ |
| 9 | 512 | −12.5 dB | −129 dBm | ✅ design centre |
| 10 | 1024 | −15 dB | −132 dBm | ✅ tight payload (8 B app) |
| 11 | 2048 | −17.5 dB | −133 dBm | ❌ exceeds 400 ms dwell |
| 12 | 4096 | −20 dB | −136 dBm | ❌ exceeds 400 ms dwell |

**Interference:** co-channel rejection against a same-SF, same-bandwidth LoRa interferer is only **−6 dB**, so an interferer 6 dB stronger than the wanted signal still kills the packet. Against a CW tone, rejection is +5 dB (SF7) to +19.5 dB (SF12).

## Bit rate

`Rs = BW / 2^SF`, `Rb = SF × Rs × 4/(4+CR)`. At 125 kHz, CR 4/5:

| SF | Symbol time | Bit rate |
|---|---|---|
| 7 | 1.024 ms | 5469 b/s |
| 9 | 4.096 ms | 1758 b/s |
| 10 | 8.192 ms | 977 b/s |

Doubling the bandwidth doubles the rate and costs 3 dB of sensitivity.

**Low Data Rate Optimize (LDRO)** is mandatory when the symbol time is ≥16 ms, i.e. SF11/SF12 at 125 kHz. It doesn't arise in this project's SF range.

## Coding rate

Cyclic FEC: 4/5 = 1.25× overhead, 4/6 = 1.5×, 4/7 = 1.75×, 4/8 = 2.0×. **4/5 is the default.** Measured at SF9 / 20 B: 185 ms (4/5) → 247 ms (4/8), a big airtime cost for a modest gain. Use 4/8 only under persistent bursty interference.

The **header is always sent at CR 4/8** with its own CRC, whatever the payload CR.

## Preamble, header and CRC: the traps

- **Preamble** defaults to 12 symbols, programmable 6–65535. **The receiver's programmed preamble length must match the transmitter's**, or the receiver's periodic preamble-detect restart drops the packet *with no interrupt raised*. This is a classic custom-protocol bug.
- **Explicit header** (default) carries the payload length, the payload CR and the CRC-present flag, and costs ~20 symbol-equivalents. **Implicit header** omits it. That saves airtime and suits fixed-size beacon or slot frames, but is bad for variable-length data.
- **The PHY CRC** is 16-bit and optional. In *implicit* mode, if TX has CRC off and RX has CRC on, the CRC always fails. In *explicit* mode the receiver must read `CrcOnPayload` before trusting `PayloadCrcError`.
- **A 16-bit CRC is not integrity for a control protocol.** It's a linear error detector on a channel that is already FEC'd. Commands that switch street lights need a keyed MAC ([[Custom-Protocol-Security]]).

## SX1276 vs SX1262: a misleading spec comparison

The widely quoted "SX126x = −148 dBm" is *not* a 125 kHz figure; it's SF12 at 7.8–10.4 kHz bandwidth. At matched 125 kHz the SX1262 is only about **1 dB** better than the SX1276. Several vendor pages compare mismatched bandwidths. Choosing the SX1276 costs almost nothing in link budget; its drawbacks are elsewhere ([[Radio-Hardware-Selection]]).

## Related

- [[LoRa-Airtime-Budget]]
- [[Link-Budget-And-Site-Survey]]
- [[Radio-Hardware-Selection]]
- [[Philippines-Regulatory-Context]]

## Sources

- Research synthesis, 2026-09-09. Semtech SX1276 datasheet Rev. 7 (Tables 10, 13, 14, 15; §4.1.1.6): https://cdn.sparkfun.com/assets/7/7/3/2/2/SX1276_Datasheet.pdf · SX1261/2 datasheet Rev 1.2: https://cdn.sparkfun.com/assets/6/b/5/1/4/SX1262_datasheet.pdf

## History

- 2026-10-14: Client specified SX1276 throughout; the Philippine radio profile (400 ms dwell) limits use to SF7–SF10. The SX1262 comparison is kept as context only.

Back to [[Index]]
