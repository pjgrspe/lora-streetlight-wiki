# LoRa vs LoRaWAN

**LoRa** is Semtech's proprietary CSS physical layer. **LoRaWAN** is the LoRa Alliance's open MAC and network layer on top of it. This project uses LoRa with a custom MAC — so every row in the right-hand column below is work you own.

## What you give up

| LoRaWAN provides | You must build |
|---|---|
| MAC framing: DevAddr, FCtrl, FCnt, FPort, FOpts, MIC | Your own frame format — [[Custom-Protocol-Frame-Design]] |
| OTAA join: DevEUI/JoinEUI/AppKey → session keys, nonce anti-replay | A pairing/provisioning protocol |
| Two-level crypto: NwkSKey (integrity) + AppSKey (confidentiality) | AES key schedule, MIC, storage, rotation — [[Custom-Protocol-Security]] |
| 32-bit frame counters with a defined replay window | Counter design, wraparound, reboot persistence |
| **ADR** — network-driven SF/power adaptation | Your own link-quality → SF controller |
| Classes A/B/C — defined downlink windows, beacons, ping slots | Your own downlink scheduling — [[Continuous-Receive-Downlink]] |
| Regional Parameters — vetted channel plans, dwell/duty enforcement, payload tables | Your own compliance logic, **and the liability** — [[Philippines-Regulatory-Context]] |
| Network + application server ecosystem, dedup, gateway diversity | Your own server-side stack |
| Interoperability and certification, off-the-shelf gateways and sensors | Nothing — you are a closed ecosystem, permanently |
| TS011 Relay (one standardised relay hop) | Your own relaying |

## When custom is justified — and it is, here

- **You need a topology LoRaWAN doesn't do.** LoRaWAN's TS011 Relay supports **exactly one relay hop**, designed for coverage infill, not as a scheduling tier. Our master ↔ slave-gateway ↔ device tree needs more. ✅
- **You need deterministic low-latency scheduled downlink to a small known node set.** Class A downlink is uplink-triggered; Class C fixes latency but requires full-time receive. A polled TDMA MAC is genuinely better for 6–10 mains-powered nodes. ✅
- **You have no LoRaWAN network server and don't want one.** The topology terminates at an RS485 local monitoring server. ✅

**Precedent:** Link Labs' Symphony Link is a commercial proprietary MAC over Semtech's LoRa PHY, built specifically to add a synchronous layer enabling repeaters and guaranteed acknowledgements. The pattern is established.

## When it becomes a mistake

If you later want 128 individually-addressed LoRa luminaires, third-party sensors, a public network fallback, or an off-the-shelf gateway — you will have built a private LoRaWAN, badly.

**Verdict: custom is justified for the stated design, but steal LoRaWAN's crypto and counter design wholesale rather than inventing it.** Read TS011 as a reference for hop addressing, security context across a relay, and relay scheduling — someone has already thought carefully about those problems.

Mitigate the lock-in at the **northbound** boundary, not the radio — see [[Northbound-Interface-Options]].

## Related

- [[LoRa-For-Control-Critique]]
- [[Master-LoRa-Gateway-Role]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://www.thethingsnetwork.org/docs/lorawan/what-is-lorawan/ · TS011 Relay: https://resources.lora-alliance.org/technical-specifications/ts011-1-0-0-relay · https://www.leverege.com/blogpost/a-primer-for-loralorawan
