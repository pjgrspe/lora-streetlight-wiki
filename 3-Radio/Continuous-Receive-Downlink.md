# Continuous Receive Is Free Here

Every node in this system is mains-powered, so its radio can listen all the time. Bottom line: **every device and gateway radio stays in continuous receive** except while transmitting. That gives sub-second downlink at no meaningful energy cost, but on the SX1276 it requires careful FIFO handling.

## Key points

- Permanent receive costs **~33 mW** on the SX1276: 0.03–0.1% of a street light's load. It's free.
- Battery-oriented LoRa practice (sleep scheduling, Class A windows) **doesn't apply**; reasoning from it makes the protocol worse.
- Continuous receive gives **sub-second unsolicited downlink**: one time-on-air (185 ms at SF9).
- **Keep the scheduled superframe for uplink anyway.** Continuous RX solves downlink; TDMA solves uplink contention.
- ⚠ **SX1276 FIFO-pointer trap in continuous RX** applies to every radio in the system. Handle it in the driver, with a dedicated host test.
- A gateway radio is "deaf" while another radio in the same box transmits ([[Dual-Radio-Redundancy-And-Coexistence]]).

## The arithmetic

| Chip | Rx current @125 kHz | Power at 3.3 V |
|---|---|---|
| **SX1276** (used here) | 9.9–11.5 mA | ~33 mW |
| SX1262 (for comparison) | 4.6 mA (Rx boosted, DC-DC) | ~15 mW |

Against a 30–100 W LED street light, 33 mW of permanent receive is **0.03–0.1%** of the luminaire's load. Even a slave with four radios listening costs ~130 mW.

> The whole body of LoRa power-optimisation practice (sleep scheduling, wake-on-radio, Class A duty cycling) **doesn't apply to these nodes**.

## Downlink latency by model

| Model | Latency | Rx duty |
|---|---|---|
| Class-A style (Rx window after each uplink) | **up to a full report period** | ~0.1% |
| Polled windows (Rx at a scheduled slot) | ½ superframe on average | superframe-dependent |
| **Class-C style (continuous Rx)** | **< 1 s** | 100% |

LoRaWAN Class C achieves sub-second downlink and is explicitly recommended for smart street lighting. The usual caveat, that Class C can't run long on batteries, is irrelevant here.

## How it's used

- **Every device and every gateway radio sits in continuous receive**, except while transmitting, or while blanked by another radio in the same box.
- A dimming command or emergency all-on reaches a node in **one time-on-air** rather than waiting for a poll slot. Group commands still carry `execute_at`, so every node switches at the same instant ([[Popcorn-Effect-And-Scheduled-Execution]]).
- On the SX1276 there's no automatic CAD-to-RX transition. Devices simply stay in RX continuous mode; CAD is used only as a pre-transmit guard ([[Media-Access-Control-Options]]).
- **The scheduled superframe is kept for uplink.** Continuous RX and TDMA are orthogonal ([[Slave-Cell-Capacity-And-Scheduling]]).

## The SX1276 FIFO trap

From the datasheet: in RX continuous mode, *"the RX modem address pointer is never reset as long as this mode is enabled. It is therefore necessary for the companion microcontroller to handle the address pointer to make sure the FIFO data buffer is never full."*

- On each `RxDone`, read `FifoRxCurrentAddr` and `RxNbBytes`, set `FifoAddrPtr` to the packet start, read the packet, and **keep the pointer handling consistent**. Sloppy handling silently corrupts later packets.
- This applies to **all** radios: 2 on the master, 4 on each slave, and one in every device. The fix lives once in the shared radio driver behind `radio_if.h`, with a host test that replays long packet sequences.

## Mains loss still has to degrade gracefully

Continuous receive is a latency optimisation, not a dependency. A node whose radio is dead must light itself from its own cached schedule ([[Fail-Safe-Lighting-Rule]]).

## Related

- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Radio-Hardware-Selection]]
- [[Media-Access-Control-Options]]
- [[Slave-Cell-Capacity-And-Scheduling]]

## Sources

- Research synthesis, 2026-09-09. https://www.thethingsnetwork.org/docs/lorawan/classes/ · SX1276 datasheet Table 10 and continuous-Rx notes · SX1261/2 datasheet Table 4-4

## History

- 2026-09-09: Recommended the SX1262's `CAD_RX` exit mode; the SX1276 FIFO trap was a caveat.
- 2026-10-14: SX1276 chosen for every radio, so the FIFO trap is now a core driver requirement; per-pole devices are the nodes listening continuously.

Back to [[Index]]
