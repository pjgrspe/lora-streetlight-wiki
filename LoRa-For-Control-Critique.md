# LoRa For Control: The Critique, And Why This Design Survives It

> **Updated 2026-10-14: the architectural line this note says to defend has been crossed.** The client's revised architecture puts a LoRa device on (apparently) every pole, ~128 radio nodes. That doesn't make the system unworkable, but the mitigations below stop being optional:
> - **Scheduled TDMA, not ALOHA or polling** ([[Slave-Cell-Capacity-And-Scheduling]])
> - **Broadcast with `execute_at`** at both tiers, to avoid the popcorn effect
> - **Autonomous astro schedules down to the device**, so the nightly switch costs zero airtime
> - The device firmware (another developer's) must implement the fail-safe and schedule fallback
>
> Device FUOTA over LoRa is now a real problem again (~12 days serial for 128 × 200 KB). It's **excluded from the v5 quote**; plan wired/bench updates for devices.

There is a real, published critique of LoRa/LoRaWAN for street lighting **control** (as opposed to monitoring). It should be taken seriously, because most of it is technically correct — and because understanding it tells you exactly which architectural line to defend.

## The critique

- **Architectural asymmetry.** LoRaWAN is designed to *receive* data, not send it. A gateway can absorb thousands of uplinks but is severely limited in downlink capacity.
- **Popcorn effect.** Group commands cause lights to activate sporadically over minutes or hours. See [[Popcorn-Effect-And-Scheduled-Execution]].
- **Duty cycle and low bandwidth** make real-time control impossible; one vendor calls LoRaWAN "significantly unsafe for public critical infrastructure in mission-critical scenarios."
- **FUOTA is brutal.** Firmware to 10,000 streetlights takes weeks or months. A 50 KB image at SF12 under a 1% duty cycle is 8–17 hours *per device*.
- **Single point of failure.** A gateway failure darkens a neighbourhood.
- **Interoperability illusion.** Devices connect, but payloads are vendor-specific, so you get lock-in anyway.

Corroborating academic work: downlink increases traffic and blocks uplink for its duration, since LoRa transceivers are half-duplex; and at a 1% duty cycle, 0.5 s of transmission locks that channel for the next 49.5 s.

(The sharpest version of the critique comes from a vendor selling a competing RF Mesh product — discount for bias, but the technical points hold.)

## Point-by-point: does it apply to us?

| Criticism | Applies here? | Why |
|---|---|---|
| Downlink asymmetry | **Much less** | This is a *scale* problem. We have 9 radio nodes, not 10,000. Downlink is cheap. |
| Popcorn effect | **Avoidable entirely** | One multicast frame + `execute_at` + a DALI broadcast per section. Three receivers, not 128. |
| Duty cycle blocking control | **Applies — must be engineered** | Airtime ledger with a hard P0 reserve; autonomous schedules so routine on/off costs zero airtime. See [[LoRa-Airtime-Budget]]. |
| "Real-time control unsafe" | **Applies to sub-second unscheduled action** | Do not promise it. Promise 2–5 s synchronised group action and be honest in the spec. |
| FUOTA impractical | **Much less** | We FUOTA 3 slaves + 6 endpoints. The 128 luminaires update over **wired DALI**. A 64 KB image to one slave at SF9 is ~1–3 h — fine in a daytime window. A strong argument for the two-tier design. |
| Gateway single point of failure | **Applies** | Mitigated only by [[Slave-Gateways-As-Intelligent-Sub-Controllers]] and [[Fail-Safe-Lighting-Rule]]. There is no other answer. |
| Interop lock-in | **Applies fully** — we are writing a custom protocol | Mitigate at the **northbound** boundary, not the radio. See [[Northbound-Interface-Options]]. |
| ISM interference / jamming | **Applies** | Multi-channel, CAD, SF escalation, noise monitoring, autonomous fallback. |

## The judgement

LoRa is a defensible choice here **precisely because the radio only carries 9 endpoints and section-level aggregates**, while wired DALI — fast and deterministic — does the last hop to 128 lights.

Had LoRa been put on every luminaire, the critique above would be close to disqualifying.

> **Do not let scope creep move per-luminaire control onto the radio.** That is the architectural line to defend.

## Deployments that do work

| Deployment | Scale | Takeaway |
|---|---|---|
| CITiLIGHT / VELOCiTI | 800,000+ streetlights, 140+ cities | LoRaWAN street lighting does work at scale, on **private** networks to avoid cellular dependency. Retrofit controllers rather than replacing fixtures. Notably, **no published latency figures** — which is itself telling. |
| Antwerp "City of Things" | City-wide | Up to 80% energy reduction in some areas; the stated lesson is economic — a shared multi-department network beats per-application networks. |
| ChirpStack community reports | Practitioner | Recurring: unexplained packet loss, 900 MHz noise, range collapsing to 50–75 m in bad RF environments. **Plan for 10x worse than datasheet.** |

## Related

- [[Master-LoRa-Gateway-Role]]
- [[Open-Questions]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://tvilight.com/lorawan-risks-in-smart-street-lighting-why-cities-are-cautious/ · https://www.semtech.com/resources/customer-stories/citilight-smart-sustainable-street-lighting-with-lorawan · https://www.iotforall.com/lorawan-strategies-for-smart-cities · https://arxiv.org/pdf/2002.01472
