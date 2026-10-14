# Continuous Receive Is Free Here

> **Updated 2026-10-14.** The client has specified the **SX1276**, so caveat 1 below (the FIFO address-pointer trap in continuous RX) **now applies to every radio in the system** — 14 gateway radios and every device. Treat it as a known driver work item with a dedicated host test. Also note the per-pole devices are the nodes that listen continuously now. See [[Slave-Cell-Capacity-And-Scheduling]].

**The nodes are mains-powered. This changes the entire downlink calculus, and most LoRa design literature does not apply.**

## The arithmetic

| Chip | Rx current @125 kHz | Power at 3.3 V |
|---|---|---|
| SX1276 | 9.9–11.5 mA | ~33 mW |
| SX1262 | 4.6 mA (Rx boosted, DC-DC) | ~15 mW |

Against a 100 W — or even a 30 W — LED street light, **15–33 mW of permanent receive is 0.03–0.1% of the luminaire's load. It is free.**

> The entire body of LoRa power-optimisation practice — sleep scheduling, wake-on-radio, Class A duty cycling, the SX1262's 600 nA sleep mode — **does not apply to this node design. Reasoning from it will make the protocol worse, not better.**

## Downlink latency by model

| Model | Latency | Rx duty |
|---|---|---|
| Class-A style (Rx window after each uplink) | **up to a full report period** | ~0.1% |
| Polled windows (Rx at a scheduled slot) | ½ superframe average | superframe-dependent |
| **Class-C style (continuous Rx)** | **< 1 s** | 100% |

LoRaWAN Class C achieves sub-second downlink and is explicitly recommended for smart street lighting; the usual caveat — "Class C devices cannot run long on batteries" — is irrelevant to us.

## Recommendation

- **Every device and slave gateway sits in Rx-continuous permanently**, except while transmitting.
- Use `cadExitMode = CAD_RX` on SX126x so a CAD detection rolls straight into reception with no host round trip.
- This gives **sub-second unsolicited downlink** — a dimming command or emergency all-on reaches a node in one time-on-air (185 ms at SF9) rather than waiting for a poll slot.
- **Keep the polled superframe anyway, for uplink.** Continuous Rx solves downlink; TDMA solves uplink contention. They are orthogonal — see [[Media-Access-Control-Options]].

## Two caveats

1. **SX127x continuous Rx has a FIFO trap.** Per the datasheet: "the RX modem address pointer is never reset as long as this mode is enabled. It is therefore necessary for the companion microcontroller to handle the address pointer to make sure the FIFO data buffer is never full." Sloppy pointer handling *will* corrupt. The SX126x buffer model is cleaner — another reason for [[Radio-Hardware-Selection]]'s recommendation.
2. **Mains loss must still degrade gracefully.** A node whose radio is dead must light itself from its own cached schedule — see [[Fail-Safe-Lighting-Rule]]. Continuous receive is a latency optimisation, not a dependency.

## Related

- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Radio-Hardware-Selection]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://www.thethingsnetwork.org/docs/lorawan/classes/ · SX1276 datasheet Table 10 and continuous-Rx notes · SX1261/2 datasheet Table 4-4
