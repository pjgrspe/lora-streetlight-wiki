# LoRa PHY Fundamentals

LoRa is Chirp Spread Spectrum: each symbol is a linear frequency sweep across the whole channel, `2^SF` chips per symbol. Because the chirp occupies the full bandwidth it resists narrowband interference, multipath and Doppler, and **demodulates below the noise floor**.

## Spreading factor

| SF | chips/symbol | Required demod SNR | Sensitivity @125 kHz (SX1276) |
|---|---|---|---|
| 7 | 128 | −7.5 dB | −123 dBm |
| 8 | 256 | −10 dB | −126 dBm |
| 9 | 512 | −12.5 dB | −129 dBm |
| 10 | 1024 | −15 dB | −132 dBm |
| 11 | 2048 | −17.5 dB | −133 dBm |
| 12 | 4096 | −20 dB | −136 dBm |

**Each +1 SF buys ~2.5 dB of link budget and costs ~2x airtime.**

> **Correction worth knowing:** the widely-quoted "SX126x = −148 dBm" is *not* a 125 kHz figure — it is SF12 at 7.8–10.4 kHz bandwidth. At matched 125 kHz the SX1262 is about **1 dB** better than the SX1276, not 11 dB. Several vendor pages compare mismatched bandwidths. Do not size a link budget from that claim.

SFs are only **quasi**-orthogonal: co-channel rejection against a same-SF, same-BW LoRa interferer is just **−6 dB**, so an interferer 6 dB stronger than your wanted signal still kills the packet. Against a CW tone, rejection is +5 dB (SF7) to +19.5 dB (SF12).

## Bit rate

`Rs = BW / 2^SF`, `Rb = SF × Rs × 4/(4+CR)`. At 125 kHz, CR 4/5:

| SF | Symbol time | Bit rate |
|---|---|---|
| 7 | 1.024 ms | 5469 b/s |
| 9 | 4.096 ms | 1758 b/s |
| 10 | 8.192 ms | 977 b/s |
| 12 | 32.77 ms | 293 b/s (250 b/s effective) |

Doubling bandwidth doubles the rate and costs 3 dB of sensitivity.

**Low Data Rate Optimize (LDRO)** is *mandatory* when symbol time ≥ 16 ms — i.e. SF11 and SF12 at 125 kHz — and steals 2 bits/symbol, which is why the official DR0/DR1 figures (250/440 b/s) are below the raw calculation.

## Coding rate

Cyclic FEC: 4/5 = 1.25x overhead, 4/6 = 1.5x, 4/7 = 1.75x, 4/8 = 2.0x. **4/5 is the right default.** Measured at SF9 / 20 B: 185 ms (4/5) → 247 ms (4/8) — double the airtime for modest gain. Only worth 4/8 under persistent bursty interference.

The **header is always sent at CR 4/8** with its own CRC regardless of payload CR.

## Preamble, header, CRC — the traps

- **Preamble**, default 12 symbols, programmable 6–65535. **The receiver's programmed preamble length must match the transmitter's**, or the receiver's periodic preamble-detect restart drops the packet *with no interrupt raised*. A classic custom-protocol bug.
- **Explicit header** (default) carries payload length, payload CR, and the CRC-present flag; costs ~20 symbol-equivalents. **Implicit header** omits it — saves airtime, mandatory for SF6, good for fixed-size beacon/slot frames, bad for variable-length data.
- **PHY CRC** is 16-bit and optional. In *implicit* mode, if TX has CRC off and RX has CRC on, the CRC is *always* wrong. In *explicit* mode the receiver must read `CrcOnPayload` before trusting `PayloadCrcError`.
- **16-bit CRC is not integrity for a control protocol.** It is a linear error detector on a channel you already FEC'd. Commands that switch a city's lights need a keyed MAC — see [[Custom-Protocol-Security]].

## Related

- [[LoRa-Airtime-Budget]]
- [[Link-Budget-And-Site-Survey]]
- [[Radio-Hardware-Selection]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. Semtech SX1276 datasheet Rev. 7 (Tables 10, 13, 14, 15; §4.1.1.6): https://cdn.sparkfun.com/assets/7/7/3/2/2/SX1276_Datasheet.pdf · SX1261/2 datasheet Rev 1.2: https://cdn.sparkfun.com/assets/6/b/5/1/4/SX1262_datasheet.pdf
