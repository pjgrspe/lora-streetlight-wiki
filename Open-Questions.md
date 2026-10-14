# Open Questions

Running log of everything unresolved, ordered by (probability × damage). Items **Q1–Q5 could each force a redesign** — they are worth chasing before any hardware or firmware commitment.

Status key: 🔴 blocking · 🟡 needed soon · 🟢 can wait · ✅ resolved

---

## Q1 🔴 Philippine radio regulation — band, power, duty cycle, licensing

**What's unknown:** NTC MC 03-08-2013's actual text was not retrievable. So the real limits on max EIRP, duty cycle, dwell time and occupied bandwidth in 915–918 MHz are unknown — and, worse, **whether the band is licence-exempt at all.** The NRFAT labels it "Point-to-Point / Point-to-Multipoint **Radio System**", phrasing that in NTC usage often implies a *licensed* service.

**Why it matters:** every airtime number in [[LoRa-Airtime-Budget]] is conditional on this. A 400 ms dwell limit **prohibits SF11 and SF12 entirely**, killing the long-range fallback. A 1% per-device duty cycle makes the 10 s poll cycle illegal at 4.5%. And if the band is licensed, that is a project-level problem, not a tuning detail.

**How to resolve:** write to NTC Equipment Standards Division (`equipment.standard@ntc.gov.ph`) asking in writing for the circular text, the licensing status, the EIRP/duty/dwell limits, and whether AS923-3 is accepted as a compliance profile.

**Decision 2026-10-14:** design to the Philippine envelope in [[Philippines-Regulatory-Context]]: **915–918 MHz**, 125 kHz channels, ≤16 dBm EIRP, ≤1% duty cycle (per box), ≤400 ms per transmission. Q1 stays 🔴 only for **licensing status and NTC's confirmed limits**, which still need the NTC letter.

*Earlier interim assumption (superseded):* 916.5–917.0 MHz, 125 kHz, ≤16 dBm EIRP, ≤1% duty cycle, ≤400 ms per transmission.

→ [[Philippines-Regulatory-Context]]

## Q2 ✅ Are the endpoint devices ours, or COTS?

**Resolved 2026-10-14: ours.** The energy meter and the LED driver/monitoring device are built in-house by the device developer on STM32 + SX1276. The protocol is ours to define. The device side implements it by linking pgorospe's device-role MAC library. See [[Team-And-Ownership]].

*Original entry:*

**What's unknown:** the "integrated LoRa energy meter" and "integrated LoRa LED driver & monitoring device" — are these our own STM32 designs (so we control the protocol) or off-the-shelf units with a fixed, possibly LoRaWAN, protocol?

**Why it matters:** **this single unknown determines whether the whole custom-protocol design is even ours to make.** If COTS, the frame format, MAC and scheduling in [[Custom-Protocol-Frame-Design]] and [[Media-Access-Control-Options]] are not our decisions, and we may be forced into master-driven polling — which breaks the airtime budget.

**How to resolve:** ask whoever is sourcing them. Get datasheets.

## Q3 ✅ DALI topology — per-pole nodes or multi-bus controllers?

**Resolved 2026-10-14: one LoRa LED driver/monitoring device per pole** (confirmed by project owner).

*Earlier reading:* **Largely answered 2026-10-14: per-pole.** The client annotation "2 [radios] for efficiency (50 devices)" implies one LoRa LED driver/monitoring device per pole, ~128 radio nodes. **Still confirm in writing**: how many luminaires does one LED driver device control? It sets the cell size and therefore the v5 quote (assumption A1). See [[Slave-Cell-Capacity-And-Scheduling]].

*Original entry:*

**What's unknown:** one DALI bus per section cannot span 1.6–2.1 km of street ([[DALI-Bus-Length-Constraint]]). The two real options are per-pole D4i controllers (**128 radio nodes**) or multi-channel DALI controllers (**6 radio nodes**).

**Why it matters:** the biggest architectural fork in the project. 128 nodes on the radio is exactly the scenario [[LoRa-For-Control-Critique]] warns about and needs scheduled TDMA across multiple channels; 6 nodes is comfortable. It also determines whether per-light fault *identity* is available at all.

**Mitigation already designed in:** make the address a hierarchical logical ID with four target types, so switching later is a mapping-table change, not a protocol change. **Do this regardless of the answer.**

## Q4 🔴 Flat (unmetered) or metered electricity service?

**What's unknown:** whether the installation is on a Meralco/co-op **flat streetlight** service (fixed assumed consumption per fixture, unmetered) or a **metered** one.

**Why it matters:** **on a flat service, dimming and CLO save zero pesos.** The entire energy-savings business case — and therefore whether the dimming logic in [[Street-Lighting-Control-Logic]] is worth building — depends on this.

→ [[Philippines-Site-Context]]

## Q5 🔴 What is the actual link budget on the real site?

**What's unknown:** everything assumes SF9 works. If foliage or terrain forces SF12, airtime rises ~7x and telemetry cadence collapses to tens of minutes — and if a 400 ms dwell limit applies (Q1), SF12 isn't even available, so the link simply fails.

**Why it matters:** unresolvable by desk research. Model error alone is ±14 dB, comparable to the whole fade margin.

**How to resolve:** a physical site survey with the real radios at the real heights, before the firmware architecture is frozen. Protocol in [[Link-Budget-And-Site-Survey]].

---

## Q6 ✅ What is the monitoring server, and who writes it?

**Resolved 2026-10-14: pgorospe writes it**, speaking Modbus. See [[Local-Monitoring-Server-Design]]. Still open: the client's PC/OS, and whether email/SMS alarm notification is wanted.

*Original entry:*

Will it tolerate quality flags, poll a job-status register, and honour asynchronous command semantics — or is it an off-the-shelf HMI that expects synchronous reads to always succeed? If the latter, the register map needs to be far more defensive, possibly with a "compatibility mode" hiding staleness behind a single comm-fail coil.

**Get the server developer into the Phase 0 spec review.** → [[Northbound-Interface-Options]], [[Cache-Behind-Synchronous-Wire-Pattern]]

## Q7 🟡 Which existing DALI drivers, and what do they actually support?

**Partially resolved 2026-09-13: the project will use existing off-the-shelf DALI drivers rather than specifying or developing them.** So the D4i procurement lever is gone, and this becomes a **verification** question with a deadline: **answer it before the register map is frozen at Gate 0.**

Still needed:

- **Make and model(s)** — and whether the population is homogeneous. Mixed models mean mixed physical minima, which causes visibly uneven dimming at the bottom of the range.
- **Are any of them D4i-certified?** Determines whether Parts 251/252/253 exist at all, and therefore whether per-light energy and diagnostics are on the table.
- **Are they already installed**, or still to be procured from an existing product line? If the latter there may still be some choice worth exercising.
- **Part 250 integrated bus PSU behaviour** — whether it's present and whether it can be disabled. More than four D4i drivers on one bus exceeds the 250 mA budget; if these are plain DALI-2 you need a **separate bus power supply** per bus instead.
- **Part 150 24 V aux** — decides whether a pole-top radio node needs its own power pack.

**Design stance meanwhile:** treat Parts 251/252/253 as optional enrichment. Nothing in the gateway's control, fault-detection or fail-safe path may depend on them, and the gateway must **probe capabilities at commissioning** rather than assume.

→ [[DALI-Telemetry-And-D4i]], [[Energy-Metering-And-Telemetry-Contract]]

## Q8 🟡 Can the LED driver device hold an autonomous fallback schedule?

Fail-safe layer 3 depends on it. If not, a dead slave gateway falls straight through to the DALI System Failure Level — an uncontrolled 100% all night. Acceptable as a last resort, poor as a second resort. Does it have an RTC, lat/long, a schedule table?

→ [[Fail-Safe-Lighting-Rule]], [[Slave-Gateways-As-Intelligent-Sub-Controllers]]

## Q9 🟡 Is the 3-phase service wye or delta?

Legacy Meralco three-phase secondary is frequently **230 V three-wire delta, not 400/230 V wye.** If the meters and the per-phase topology assume a 4-wire wye with neutral and the service is 3-wire delta, **the L-N voltage readings and per-phase power calculations will be wrong.** Confirm on site before interpreting meter registers. → [[Philippines-Site-Context]]

## Q10 🟡 Will NTC accept the RAK3172's existing FCC/CE certification?

Determines whether the pre-certified-module route actually saves the months it promises, or whether local testing is required anyway. **Worth several months of schedule.** → [[Radio-Hardware-Selection]]

## Q11 🟡 STM32WLE5 live price and lead time

The WL55 line was quoted at a **30-week factory lead time** beyond stock. A live WLE5CCU6 quote was not obtainable. If the lead time is comparable, take the STM32G0B1 + SX1262 fallback — which is arguably the better engineering choice anyway (dual-bank flash, 4x flash, 6 UARTs). → [[STM32-Part-Selection]]

## Q12 🟡 DOE / DPWH mandated lux and uniformity by road class

The primary DOE 2017 Roadway Lighting Guidelines PDF was not retrievable (host DNS failure). Circulating figures are secondary. **This legally bounds how deep you may dim** — MIN LEVEL and dimming profiles must not take illuminance below the mandate. → [[Street-Lighting-Control-Logic]]

## Q13 🟡 DALI-2 frame settling timings

Sources conflict: forward→backward 2.92–9.17 ms vs 5.5–10.5 ms; forward→forward ≥9.17 ms vs >13.5 ms — probably a DALI-1 vs DALI-2 difference, unresolvable without the paid IEC 62386-101 Ed.3. **Interim: wait the union** (receive window 2.9–11 ms, forward spacing ≥13.5 ms). → [[DALI-Fundamentals]]

---

## Q14 🟢 Duty-cycle accounting across a two-hop relay

A slave is both receiver and transmitter for the *same* logical message. Whose airtime budget does a relayed frame charge to? No authoritative treatment for private multi-hop LoRa was found; TS011 is the closest reference. **Design the ledger per-radio, not per-message**, and instrument it so Gate 2 can prove compliance. → [[LoRa-Airtime-Budget]]

## Q15 🟢 Outage buffer sizing

How many events per day, and how long an outage must the gateway survive? "7 days of full telemetry" needs external NOR, not internal flash. Depends on Q2's telemetry volume. **Likely answer: downsample** — 15-minute summaries plus full resolution for the last hour. → [[Persistence-And-Storage]]

## Q16 🟢 Physical distances between master, sub-gateways and devices

Not stated anywhere in the diagram, and it feeds directly into Q5. Needed for the survey plan.

## Q17 🟢 Is VBAT bonded out on the chosen package/module?

Some modules tie VBAT to VDD internally, making RTC backup impossible and forcing an external RTC. Also: confirm the RTC backup-register count on STM32WL (assumed 20). → [[Time-Distribution-And-Clock-Trust]]

## Q18 🟢 Scale-out intentions

If this pilot becomes 10 sections and 1000 lights, does the single-radio slotted design still hold? Probably to ~10 slaves, but the 3 MHz spectrum limit and the master's duty cycle become binding. **Worth knowing now so the address space and register-map strides don't need renumbering later.**

## Q19 🟢 Security posture expectations

Is this a municipal asset with a security review in its future? Retrofitting a MIC and frame counters after 128 lights are live is expensive and will therefore be skipped — **so put them in the Phase 0 wire format even if key management stays crude at pilot.** → [[Custom-Protocol-Security]]

## Q20 🟢 Semtech AN1200.48 CAD parameters

Referenced by the SX1262 datasheet but not publicly retrievable. `cadDetPeak`/`cadDetMin` will need empirical tuning or a Semtech NDA. → [[Media-Access-Control-Options]]

---

## Added 2026-10-14 (revised architecture)

## Q21 🔴 Does a multi-radio box get one duty-cycle budget or one per radio — and are ≥8 channels allowed?

A slave now has 4 radios and the system needs ≥8 channels across 915–918 MHz. The interim regulatory envelope (916.5–917.0 MHz) fits only ~3. **Add both questions to the NTC letter (Q1).** Meanwhile, design the airtime ledger per box. → [[Dual-Radio-Redundancy-And-Coexistence]]

## Q22 ✅ Who designs and builds the gateway hardware?

**Resolved 2026-10-14:** a separate gateway hardware developer. pgorospe develops on client-supplied STM32 dev boards + SX1276 modules, and the final gateway hardware is expected to follow the same MCU and radio. pgorospe supplies a **gateway hardware requirements spec** at M1 (radio wiring, separate SPI bus per radio of a pair, antenna separation, RS485, RTC backup). ⚠ Still to learn: **which exact STM32 dev board** (pin and SPI count for 4 radios).

*Original entry:*

The old team had a hardware developer. The revised team has only pgorospe + the device developer. "We're gonna use STM boards" suggests off-the-shelf STM32 boards plus SX1276 modules. Which boards exactly (Nucleo? Blue Pill? a client in-house board?), and who owns the enclosure, power, surge protection and antennas? **The v5 quote assumes client-supplied dev boards + modules, with custom PCB design as an optional add-on.** → [[STM32-Part-Selection]]

## Q23 🟡 SX1276 is "not recommended for new designs" — is the client aware?

Semtech's own pages carry that status. It's not end-of-life, but it's a supply risk across ~150 radios. Recommend a buffer buy, and keeping the radio abstraction so SX1262 stays a port rather than a rewrite. → [[Radio-Hardware-Selection]]

## Q24 🟡 Measured antenna isolation inside the real enclosure

All in-box coexistence rules assume ~20–30 dB isolation, which is far short of what simultaneous TX/RX needs. Measure it in Phase 2. → [[Dual-Radio-Redundancy-And-Coexistence]]

## Q25 ✅ Does each device have one radio or two?

**Resolved 2026-10-14: one.** If devices also carry two radios, the device-side MAC library and the cell failover design change.

## Q26 🟢 How many real devices will the device developer supply for bench integration, and when?

The v5 quote assumes at least 1 meter + ~6 LED devices by the start of integration, with the rest of the 54-device cell simulated.

## Q27 🟡 How many sections / Slave Gateways?

**Raised 2026-10-14:** the diagram shows 3, but the project owner says it may be 4 or more. Client documents no longer state a number. Design for N: one Slave Gateway firmware, a tier-1 slot schedule sized from the slave count in config, and a channel plan of 2 tier-1 channels + 2 per section, which grows with N and so depends even more on Q1/Q21. The per-section ceiling stays at 60 devices.

---

## Things worth verifying against primary documents

Flagged where research relied on secondary sources or fetches failed:

- STM32WLE5 datasheet RX sensitivity table and flash endurance (fetches timed out)
- AN5554 FUOTA flash partitioning
- Modbus-over-serial spec: the 150 Ω termination, 450–650 Ω polarisation range, and 1000 m-at-9600 figures
- DALI-2 `System Failure Level` / `Power-On Level` exact semantics and the ~550 ms interruption threshold (secondary sources only)
- Coax loss figures per metre (typical manufacturer values, not datasheet-confirmed)
- Mean Well IRM series IEC 61000-4-5 surge immunity (not in summary specs)

## Related

- [[Development-Roadmap-And-Gates]] — Gate 0 is where most of these should be closed
- [[Index]]
