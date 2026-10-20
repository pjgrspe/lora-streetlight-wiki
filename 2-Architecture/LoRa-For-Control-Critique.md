# LoRa For Control: The Critique, And Why This Design Survives It

There is a real, published critique of LoRa/LoRaWAN for street lighting **control** (as opposed to monitoring), and most of it is technically correct. The original design dodged it by keeping LoRa off the luminaires. **The revised design puts a LoRa device on every pole, so the critique now applies in full**, and the mitigations below stop being optional.

## Key points

- The architectural line this note once defended, "don't put per-luminaire control on the radio", **has been crossed**: there's one LoRa device per pole.
- That's workable, but only with these mitigations as hard requirements: **scheduled TDMA** (not ALOHA or polling), **broadcast + `execute_at`** at both tiers, **autonomous schedules down to the device**, and a **device-level fail-safe** in device firmware.
- Two-hop topology, dual radios per gateway link, and LoRa only between gateways and devices keep the radio problem bounded per section (up to 60 devices per Slave).
- Device firmware update over LoRa is a hard problem again, and it's excluded from the current engagement. Plan wired/bench updates for devices.

## The critique

- **Architectural asymmetry.** LoRaWAN is designed to *receive* data; gateways absorb many uplinks but have little downlink capacity.
- **Popcorn effect.** Group commands make lights switch on sporadically over minutes or hours ([[Popcorn-Effect-And-Scheduled-Execution]]).
- **Duty cycle and low bandwidth** make real-time control impossible. One vendor calls LoRaWAN "significantly unsafe for public critical infrastructure in mission-critical scenarios."
- **FUOTA is brutal.** A 50 KB image at SF12 under a 1% duty cycle is 8–17 hours *per device*.
- **Single point of failure.** A gateway failure darkens a neighbourhood.
- **Interoperability illusion.** Devices connect, but payloads are vendor-specific.

Corroborating academic work: LoRa transceivers are half-duplex, so downlink blocks uplink while it runs; and at a 1% duty cycle, 0.5 s of transmission locks that channel for the next 49.5 s. (The sharpest version of the critique comes from a vendor selling a competing RF mesh product. Discount for bias, but the technical points hold.)

## Point by point, for this design

| Criticism | Applies? | How it's handled |
|---|---|---|
| Downlink asymmetry | **Yes, per cell** | Devices listen continuously (mains-powered); the Slave's downlink is one beacon per channel per cycle (carrying the event-ACK bitmap) plus broadcast commands, not per-device polls ([[Slave-Cell-Capacity-And-Scheduling]]) |
| Popcorn effect | **Yes, now that each pole has a radio** | One broadcast with `execute_at` per tier: master → Slaves, then each Slave → its cell on both channels |
| Duty cycle blocking control | **Yes, must be engineered** | Airtime ledger per box with a hard P0 reserve; scheduled uplinks keep the Slave's own TX low; autonomous schedules make the nightly switch cost zero airtime ([[LoRa-Airtime-Budget]]) |
| "Real-time control unsafe" | **Yes, for sub-second unscheduled action** | Don't promise it. Promise synchronised group action within seconds, and say so in the spec |
| FUOTA impractical | **Yes again, for devices** | ~12 days serial for 128 × 200 KB. Excluded from the current engagement; devices are updated wired/on the bench. Gateway OTA stays feasible ([[Firmware-Update-Over-LoRa]]) |
| Gateway single point of failure | **Yes** | Autonomy at every level ([[Slave-Gateways-As-Intelligent-Sub-Controllers]], [[Fail-Safe-Lighting-Rule]]). Dual radios cover radio/antenna faults, not box death |
| Interop lock-in | **Yes**: the protocol is custom | Mitigate at the **northbound** boundary, not the radio ([[Northbound-Interface-Options]]) |
| ISM interference / jamming | **Yes** | Paired radios on separate channels (frequency diversity), CAD, failover to the standby channel or single-channel mode, noise monitoring, autonomous fallback |

## The judgement

Per-pole LoRa is defensible here **because it is bounded**: a two-tier topology, at most 60 devices per Slave on two channels, scheduled TDMA instead of ALOHA, and every node able to keep the lights correct on its own. Drop any of those and the critique becomes close to disqualifying again.

> **The line to defend now:** never let the nightly on/off, or the fail-safe, depend on the radio network.

## Deployments that do work

| Deployment | Scale | Takeaway |
|---|---|---|
| CITiLIGHT / VELOCiTI | 800,000+ streetlights, 140+ cities | Per-luminaire LoRaWAN control works at scale on **private** networks. Retrofit controllers rather than replacing fixtures. Notably, **no published latency figures**, which is itself telling |
| Antwerp "City of Things" | City-wide | Up to 80% energy reduction in some areas; the lesson is economic: a shared multi-department network beats per-application networks |
| ChirpStack community reports | Practitioner | Recurring: unexplained packet loss, 900 MHz noise, range collapsing to 50–75 m in bad RF environments. **Plan for 10× worse than the datasheet** |

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Media-Access-Control-Options]]
- [[Open-Questions]]

## Sources

- Research synthesis, 2026-09-09. https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/ · https://www.semtech.com/resources/customer-stories/citilight-smart-sustainable-street-lighting-with-lorawan · https://www.iotforall.com/lorawan-strategies-for-smart-cities · https://arxiv.org/pdf/2002.01472

## History

- 2026-09-09: written when the radio carried ~9 nodes and DALI did the last hop; the advice was "don't put per-luminaire control on the radio".
- 2026-10-14: the revised architecture put a LoRa device on every pole; mitigations became requirements.
- 2026-10-21: aligned with spec pack v0.1 (no SF escalation in v1; interference handled by channel failover).

Back to [[Index]]
