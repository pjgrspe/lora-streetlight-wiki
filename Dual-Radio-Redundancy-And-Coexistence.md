# Dual-Radio Redundancy And Coexistence

How the paired LoRa radios work: two on the master (both pointing at the slaves), and four on each slave (two upstream to the master, two downstream to the devices). The pairs give **redundancy, plus a second receive channel**. They do **not** let you transmit and receive at the same time inside one box.

Introduced 2026-10-14 with the revised architecture. See [[System-Architecture-As-Diagrammed]].

## Radio count

| Box | Upstream pair | Downstream pair | Total |
|---|---|---|---|
| Master | — | 2 (to the 3 slaves) | **2** |
| Slave ×3 | 2 (to master) | 2 (to ~38–54 devices) | **4 each** |
| Devices | 1 | — | 1 each |

14 gateway radios plus up to ~140 device radios (exact count TBC), all SX1276 ([[Radio-Hardware-Selection]]).

## The physics constraint: radios in one box blind each other

Two radios in the same enclosure, in the same 3 MHz band, are a few tens of centimetres apart.

| Quantity | Value |
|---|---|
| TX power (SX1276, conservative legal setting) | +14 to +17 dBm |
| Antenna-to-antenna isolation, 2 dBi whips, free-space approximation | ~17 dB at 0.3 m · ~22 dB at 0.5 m · ~28 dB at 1 m |
| Interferer arriving at the other radio | **≈ −11 to +0 dBm** |
| SX1276 blocking tolerance at 2 MHz offset (94 dB above SF12 sensitivity, datasheet figure via secondary source) | **≈ −40 dBm** |
| Shortfall | **~30–40 dB** |

> **When any radio in a box transmits, every other radio in that box must be treated as deaf.** That holds even with the whole 3 MHz band between the channels. Vertical antenna separation (stacked collinear) adds maybe 10–20 dB, which is still not enough to plan around.

⚠ The 94 dB blocking figure comes from a secondary citation of the datasheet ([arXiv 2011.03952](https://arxiv.org/pdf/2011.03952)), and the isolation figures are far-field estimates for what is really near-field. **Measure isolation on the real enclosure in Phase 2.** If the measured isolation is very good, the rule can be relaxed. Don't count on it.

### Design rule that follows

- **One transmit scheduler per box, owning all its radios.** A TX on any radio opens a "blanking" window in which the other radios' RX is discarded and isn't counted as link loss.
- **Receiving on several radios at once is fine.** Both downstream radios listening on two channels at the same time is what doubles cell uplink capacity. See [[Slave-Cell-Capacity-And-Scheduling]].
- **The slave's two tiers must be time-aligned.** When the slave transmits upstream to the master, its downstream radios are blind. So the master's tier-1 slot for slave *n* must fall inside a quiet period of slave *n*'s cell superframe. **The master superframe and the cell superframe have to be designed together**, as one timing plan.

## What the second radio is for

### Master ↔ slave pair (tier 1)

Tier-1 traffic is tiny (3 slaves, summaries and exceptions), so **the second radio adds nothing for capacity here. Its job is availability.**

Recommended mode: **active / exercised-standby**, not cold standby.

| Radio | Channel | Carries |
|---|---|---|
| A — primary | e.g. 915.3 MHz | All scheduled tier-1 traffic |
| B — secondary | e.g. 917.7 MHz (opposite edge of the band, for frequency diversity) | **A heartbeat/beacon every superframe, so its health is always proven.** Also carries a **duplicate of every P0 command** (emergency ON, `execute_at` group commands) |

- **A backup that is never exercised is a latent failure.** If radio B only gets used when A dies, its dead coax or failed connector is discovered at the worst moment. Heartbeating on both, every cycle, catches that.
- **Duplicating P0 on both channels is cheap** (a few frames per day) and adds frequency diversity against narrowband interference on the critical path. Receivers deduplicate on `SEQ` ([[Custom-Protocol-Frame-Design]]).
- **Failover:** if A misses *N* consecutive heartbeats, or its PER crosses a threshold, B takes all traffic. Failback only after A has passed a hold-off period (e.g. 10 minutes clean) to avoid flapping. Both events raise an alarm.
- **Load-sharing (both carrying traffic all the time) is possible but not worth it** at tier 1. It doubles the state to debug and buys nothing, because tier 1 isn't busy.

### Slave ↔ device pair (tier 2)

Here the second radio **does** add capacity, because ~54 devices share it. See [[Slave-Cell-Capacity-And-Scheduling]].

- **Both radios active, splitting the devices.** Each device is homed to channel A or channel B at join, roughly half each. Both radios receive concurrently.
- The slave sends its cell beacon on **both** channels.
- **Failover is device-driven:** a device that misses *N* beacons on its home channel scans to the other channel and re-registers. The surviving radio then serves all ~54 devices. At SF9 that is ~15 s of uplink per cycle, which still fits a 60 s cycle ([[Slave-Cell-Capacity-And-Scheduling]]), so **a failed downstream radio halves capacity but doesn't lose the cell**.

## What dual radios protect against, and what they don't

Be honest about this in the client documentation:

| Failure | Covered by the second radio? |
|---|---|
| Radio module failure | ✅ |
| Antenna, coax or connector failure, including water ingress (**the most common slow-degradation failure in tropical installs**) | ✅ if each radio has its **own antenna and own coax run** |
| Narrowband interference on one channel | ✅ (frequency diversity) |
| One SPI bus jammed by a failed radio (e.g. stuck MISO) | ✅ **only if the pair sits on separate SPI buses** |
| MCU crash, PSU failure, lightning damage to the box | ❌ the box is still a single point of failure. That is handled by autonomy ([[Slave-Gateways-As-Intelligent-Sub-Controllers]], [[Fail-Safe-Lighting-Rule]]) |

**Hardware rules for real independence:** the primary and backup radio get **separate SPI buses, separate reset lines, separate LDOs (or at least separately fused rails), separate antennas and separate coax**. Two radios sharing one SPI bus give you about half the redundancy you think you're paying for.

## ⚠ Regulatory open point

If the Philippine rule is a per-*device* duty cycle ([[Philippines-Regulatory-Context]]), it's unclear whether a 4-radio slave counts as one device (one shared 1% budget) or four. **Design the airtime ledger per box, not per radio** (the conservative reading) until NTC answers. Tracked in [[Open-Questions]] Q21.

The channel plan also needs **at least 8 channels** (2 tier-1 + 2 per section × 3). That needs the **full 915–918 MHz** slice. The old interim assumption of a 916.5–917.0 MHz core fits only ~3 channels. This makes Q1 even more critical.

## Related

- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Media-Access-Control-Options]]
- [[Radio-Hardware-Selection]]
- [[Failure-Modes-And-Mitigations]]
- [[Index]]

## Sources

- Client information relayed by project owner, 2026-10-14: 2 LoRa radios master↔slave ("1 is backup"), 2 slave↔devices ("for efficiency, ~50 devices"), 4 per slave in total.
- Semtech SX1276 product pages, "not recommended for new design" status: https://www.semtech.com/products/wireless-rf/lora-connect/sx1276dvk1ias · https://forum.lora-developers.semtech.com/t/sx1276-nrfnd/1265
- SX1276 2 MHz blocking figure (94 dB, SF12/125 kHz) as cited in https://arxiv.org/pdf/2011.03952
- Isolation figures: free-space path loss at 916.5 MHz minus 2 × 2 dBi, computed 2026-10-14. Near-field effects are not modelled.
