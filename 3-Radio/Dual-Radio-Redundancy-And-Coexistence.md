# Dual-Radio Redundancy And Coexistence

How the paired LoRa radios on each gateway work: two on the master, four on each slave (two upstream, two downstream). Bottom line: the pairs give **redundancy, plus a second receive channel**. They do **not** let a box transmit and receive at the same time, so each box needs a single transmit scheduler.

## Key points

- **Master:** 2 radios toward the slaves, a primary plus an **exercised standby** that heartbeats every cycle. P0 commands go out on both, repeated once.
- **Slave:** 2 radios up to the master (same scheme), and 2 down to its devices, **both active** on separate channels, each serving about half the devices.
- **A transmitting radio deafens every other radio in the same box.** One TX scheduler per box, with blanking windows. The master's tier-1 slot for each slave falls inside that slave's cell quiet window.
- **Real independence needs separate SPI buses, resets, power rails, antennas and coax** for the two radios of a pair.
- Dual radios cover radio, antenna, coax and narrowband-interference failures, **not loss of the whole box**. That's handled by autonomy and the fail-safe rule.
- Channel plan: **2 tier-1 channels at opposite band edges + 2 per section**, inside 915–918 MHz. Airtime ledger **per box**.

## Radio count

| Box | Upstream pair | Downstream pair | Total |
|---|---|---|---|
| Master | — | 2 (to all slaves) | **2** |
| Each Slave (one per section) | 2 (to master) | 2 (to its devices, up to 60) | **4** |
| Each device | 1 | — | **1** |

All radios are SX1276 ([[Radio-Hardware-Selection]]). The number of sections isn't fixed, so the gateway radio count is `2 + 4 × sections`.

## The physics: radios in one box blind each other

Two radios in the same enclosure, in the same 3 MHz band, sit a few tens of centimetres apart.

| Quantity | Value |
|---|---|
| TX power (SX1276, at or below the legal 16 dBm EIRP) | +14 to +17 dBm |
| Antenna-to-antenna isolation, 2 dBi whips, free-space approximation | ~17 dB at 0.3 m · ~22 dB at 0.5 m · ~28 dB at 1 m |
| Interferer arriving at the other radio | **≈ −11 to 0 dBm** |
| SX1276 blocking tolerance at 2 MHz offset (94 dB above SF12 sensitivity) | **≈ −40 dBm** |
| Shortfall | **~30–40 dB** |

> **When any radio in a box transmits, every other radio in that box must be treated as deaf**, even with the whole 3 MHz band between their channels. Vertical antenna separation adds maybe 10–20 dB, which is still not enough to plan around.

⚠ The 94 dB blocking figure comes from a secondary citation of the datasheet, and the isolation figures are far-field estimates for what is really near-field. **Measure isolation on the real enclosure on the bench** ([[Open-Questions]] Q24). If it turns out much better, the rule can be relaxed. Don't count on it.

### Design rules

- **One transmit scheduler per box, owning all its radios.** A TX on any radio opens a blanking window: the other radios' RX during it is discarded and isn't counted as link loss.
- **Receiving on several radios at once is fine.** Both downstream radios listening on two channels at once is what doubles cell uplink capacity ([[Slave-Cell-Capacity-And-Scheduling]]).
- **The slave's two tiers are time-aligned.** When a slave transmits to the master, its downstream radios are blind. So the master's tier-1 slot for slave *n* sits inside slave *n*'s cell quiet window, and the two superframes are designed as one timing plan.

## What the second radio is for

### Master ↔ slave pair (tier 1)

Tier-1 traffic is small (summaries and exceptions per section), so **the second radio is for availability, not capacity.**

| Radio | Channel | Carries |
|---|---|---|
| A, primary | **ch 0, 915.2 MHz** (lowest band edge) | All scheduled tier-1 traffic |
| B, exercised standby | **ch 13, 917.8 MHz** (opposite edge), for frequency diversity | **A heartbeat every cycle** from the master (every 5th cycle from each slave), so its health is always proven, plus **a duplicate of every P0 command** (emergency ON, `execute_at` group commands) |

- **A backup that is never exercised is a latent failure.** If radio B is only used when A dies, its dead coax is discovered at the worst moment. Heartbeating on both every cycle catches that.
- **Duplicating P0 on both channels is cheap** (a few frames a day). Receivers deduplicate on `SEQ` ([[Custom-Protocol-Frame-Design]]).
- **Failover:** after **3 consecutive missed exchanges** on A (or a failed radio self-test), B takes all traffic: the beacon moves to ch 13 and the heartbeat to ch 0, so A keeps being exercised. **Failback** only after A has been clean for **5 cycles (10 min)**, to avoid flapping. Both raise an alarm. Slots, timing and addresses don't change, only the channel.
- **A dead standby is detected too:** no heartbeat heard on it for 3 cycles at a slave, or 10 at the master (each slave only sends one every 5th cycle), raises `STANDBY_RADIO_DOWN`.
- **Load-sharing** (both radios carrying traffic all the time) doubles the state to debug and buys nothing on a lightly loaded tier.

### Slave ↔ device pair (tier 2)

Here the second radio **does** add capacity, because up to 60 devices share it.

- **Both radios active, splitting the devices.** Each device is assigned to channel A or B at join, roughly half each, and both radios receive concurrently.
- The slave sends its cell beacon on **both** channels.
- **Failover is device-driven:** a device that misses **2 beacons** on its home channel listens on the other one. If that beacon shows `single_channel_mode`, the device keeps its slot number there with no rejoin. The surviving radio runs all 64 slot positions back to back (19.2 s), still well inside the 120 s cycle, and the slave shifts its cell cycle 10 s earlier so its tier-1 slot stays in the quiet window ([[Slave-Cell-Capacity-And-Scheduling]]). **A failed downstream radio halves capacity but doesn't lose the cell.**

## What dual radios protect against

| Failure | Covered? |
|---|---|
| Radio module failure | ✅ |
| Antenna, coax or connector failure, including water ingress (**the most common slow-degradation failure in tropical installs**) | ✅ if each radio has its **own antenna and coax run** |
| Narrowband interference on one channel | ✅ frequency diversity |
| A failed radio jamming its SPI bus (e.g. stuck MISO) | ✅ **only if the pair is on separate SPI buses** |
| MCU crash, PSU failure, lightning damage to the box | ❌ the box is still a single point of failure. Covered by autonomy ([[Slave-Gateways-As-Intelligent-Sub-Controllers]], [[Fail-Safe-Lighting-Rule]]) |

**Hardware rules for real independence:** the two radios of a pair get **separate SPI buses, separate reset lines, separate LDOs (or at least separately fused rails), separate antennas and separate coax**. Two radios sharing one SPI bus give about half the redundancy you think you're paying for. These rules belong in the gateway hardware requirements handed to the hardware developer ([[STM32-Part-Selection]]).

## Regulatory fit

The project designs to the Philippine envelope: 915–918 MHz, 125 kHz channels, ≤16 dBm EIRP, ≤400 ms per TX, ≤1% duty ([[Philippines-Regulatory-Context]]).

- **Channels:** the 200 kHz raster gives **14 channels** (915.2–917.8 MHz). Tier 1 uses ch 0 and ch 13. Section n uses **ch n and ch n+6**, 1.2 MHz apart so a slave's two receivers aren't on adjacent channels. Up to **6 sections** fit before pairs must be reused, which then depends on physical separation (spec 04).
- **Duty cycle:** it's unclear whether a 4-radio slave counts as one device (one shared 1% budget) or four. **The airtime ledger is kept per box** (the conservative reading) until NTC answers ([[Open-Questions]] Q21).

## Related

- [[Slave-Cell-Capacity-And-Scheduling]]
- [[Media-Access-Control-Options]]
- [[Radio-Hardware-Selection]]
- [[Failure-Modes-And-Mitigations]]
- [[LoRa-Airtime-Budget]]

## Sources

- `Specs/specs/02-tier1-protocol.md` §12, `03-tier2-device-protocol.md` §11, `04-channel-plan-and-airtime.md` (v0.1, 2026-10-21)
- Client information relayed by the project owner, 2026-10-14: 2 LoRa radios master↔slave ("1 is backup"), 2 slave↔devices ("for efficiency, ~50 devices"), 4 per slave in total.
- Semtech SX1276 product pages, "not recommended for new design" status: https://www.semtech.com/products/wireless-rf/lora-connect/sx1276dvk1ias · https://forum.lora-developers.semtech.com/t/sx1276-nrfnd/1265
- SX1276 2 MHz blocking figure (94 dB, SF12/125 kHz) as cited in https://arxiv.org/pdf/2011.03952
- Isolation figures: free-space path loss at 916.5 MHz minus 2 × 2 dBi, computed 2026-10-14. Near-field effects are not modelled.

## History

- 2026-10-14: Note created when the client specified dual radios on every gateway link.
- 2026-10-14: Number of sections no longer fixed (channel need generalised to 2 + 2 per section); Philippine radio profile adopted.
- 2026-10-21: aligned with spec pack v0.1. Concrete channels (tier 1 ch 0/ch 13, section n ch n/ch n+6, 14 channels); failover after 3 missed exchanges, failback after 5 clean cycles; standby-down detection; device moves channel after 2 missed beacons; single-channel mode shifts the cell cycle.

Back to [[Index]]
