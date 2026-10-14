# LoRa vs LoRaWAN

Why this project runs its own MAC on the LoRa radio instead of using LoRaWAN, and what that costs. Bottom line: a custom MAC is justified for this two-tier, dual-radio, time-slotted topology, but copy LoRaWAN's crypto and counter design rather than inventing your own.

## Key points

- **LoRa** is Semtech's Chirp Spread Spectrum physical layer. **LoRaWAN** is the LoRa Alliance's MAC and network layer on top of it. This project uses LoRa with a **custom MAC**.
- Everything LoRaWAN would provide (framing, join, crypto, counters, data-rate adaptation, regional compliance) is **work we own**.
- **Justified here** because LoRaWAN can't do a scheduled master → slave → device tree with dual radios, and the site has no network server.
- With a LoRa device on every pole, the system is now LoRaWAN-sized. **The risk is building "a private LoRaWAN, badly."** Mitigate by borrowing LoRaWAN's security design wholesale.
- Lock-in is handled at the **northbound** boundary (the server), not the radio.

## What you give up

| LoRaWAN provides | We build instead |
|---|---|
| MAC framing: DevAddr, FCtrl, FCnt, FPort, FOpts, MIC | Our own frame format: [[Custom-Protocol-Frame-Design]] |
| OTAA join: DevEUI/JoinEUI/AppKey → session keys, nonce anti-replay | A pairing and provisioning protocol |
| Two-level crypto: NwkSKey (integrity) + AppSKey (confidentiality) | AES key schedule, MIC, storage, rotation, in software on our MCUs: [[Custom-Protocol-Security]] |
| 32-bit frame counters with a defined replay window | Counter design, wraparound, reboot persistence |
| **ADR**: network-driven SF and power adaptation | Our own link-quality → SF controller (capped at SF10) |
| Classes A/B/C: defined downlink windows, beacons, ping slots | Our own downlink and slot scheduling: [[Continuous-Receive-Downlink]], [[Slave-Cell-Capacity-And-Scheduling]] |
| Regional Parameters: vetted channel plans, dwell/duty enforcement, payload tables | Our own compliance logic, **and the liability**: [[Philippines-Regulatory-Context]] |
| Network and application server ecosystem, dedup, gateway diversity | Master Gateway + Local Monitoring Server |
| Interoperability, certification, off-the-shelf gateways and sensors | Nothing; a closed ecosystem, permanently |
| TS011 Relay (one standardised relay hop) | Our own two-tier relaying |

## Why custom is justified here

- **The topology isn't one LoRaWAN supports.** TS011 Relay allows **exactly one relay hop**, designed for coverage infill, not as a scheduling tier. We need master → slave → device, with slaves acting as autonomous sub-controllers ✅.
- **We need deterministic scheduling across dual radios and multiple channels.** LoRaWAN uplink is essentially ALOHA, which can't serve ~54 devices per cell reliably ([[Media-Access-Control-Options]]). Our scheduled-uplink TDMA cell plus a coordinated in-box transmit schedule has no LoRaWAN equivalent ✅.
- **There is no LoRaWAN network server, and we don't want one.** The system terminates at a Modbus RS485 link to a local server ✅.

**Precedent:** Link Labs' Symphony Link is a commercial proprietary MAC over Semtech's LoRa radio, built to add a synchronous layer with repeaters and guaranteed acknowledgements. The pattern is established.

## Where it could become a mistake

The project now has a LoRa device on every pole: well over a hundred radio nodes. That is exactly the scale at which a custom MAC starts duplicating LoRaWAN. It's still justified by the two-tier scheduled topology, but:

- **Copy LoRaWAN's crypto, counter and join design wholesale** (1.1 behaviour) rather than inventing them.
- Read **TS011** as a reference for hop addressing, security context across a relay, and relay scheduling.
- If the project later wants third-party sensors, a public network fallback or off-the-shelf gateways, revisit this decision.

## Related

- [[LoRa-For-Control-Critique]]
- [[Master-LoRa-Gateway-Role]]
- [[Custom-Protocol-Security]]
- [[Northbound-Interface-Options]]

## Sources

- Research synthesis, 2026-09-09. https://www.thethingsnetwork.org/docs/lorawan/what-is-lorawan/ · TS011 Relay: https://resources.lora-alliance.org/technical-specifications/ts011-1-0-0-relay · https://www.leverege.com/blogpost/a-primer-for-loralorawan

## History

- 2026-09-09: Justified custom MAC for a 6–10 node polled network.
- 2026-10-14: Per-pole devices made the network LoRaWAN-sized; justification now rests on the two-tier scheduled topology and dual radios.

Back to [[Index]]
