# Commissioning And Field Swap

Commissioning is the controlled process of proving each installed node is correctly identified, located, enrolled, configured and accepted before operations takes ownership. It succeeds through a **repeatable workflow with clear exception records** — not by treating every pole as an isolated installation.

## Join model: hybrid

| Model | Pros | Cons |
|---|---|---|
| Fully pre-provisioned (keys + address burned at the bench) | Simple, no join protocol, no join-time attack window | Every field swap needs a bench trip; address bookkeeping errors |
| Open over-the-air pairing | Trivial field swap | Anyone with a $10 radio joins your lighting network |
| **Hybrid — recommended** | **Root key + UID pre-provisioned at the bench; short address and session keys assigned OTA, only during a time-boxed, physically authorised `COMMISSIONING` window** | A modest amount of protocol work |

Key derivation: `device_key = KDF(site_master_key, UID)`. You then never store per-device keys in the field — only the site key in the master, in a protected flash region. See [[Custom-Protocol-Security]].

## Site survey — do this before the enclosure is designed

1. **Survey with the actual radios, at the actual heights, with the actual antennas.** Simulations lie; range collapse from a theoretical 15 km to a real 2–5 km urban figure is the norm.
2. Record RSSI/SNR/PER at every planned slave location, at both the intended working SF and the fallback SF — and record the **noise floor separately**, with all your transmitters off.
3. **Soak each link 24 h+**, not a five-minute spot check. Catch diurnal interference and thermal effects. Take at least one measurement in rain.
4. Acceptance: **≥15 dB margin at the working SF**, with ~25 dB as the design target in vegetated terrain.
5. Document antenna type, height, polarisation, cable length and loss, and orientation photos per site. **Connector water ingress is the most common slow-degradation failure in tropical installs.**
6. Scan for existing occupancy in the band before committing to channels — see [[Philippines-Regulatory-Context]].

## Field device swap

Target: **one technician, no laptop, under 10 minutes, no server-side reconfiguration.**

```
1. Server/UI (or a button+LED on the master): "REPLACE device at asset PH-A-METER-01"
   → master opens a targeted commissioning window (5 min, one slot only, logged)
2. Tech installs the new unit and powers it up.
3. New unit joins with its UID + root key. Master validates the device type matches,
   then REBINDS the existing short address, asset ID, config, thresholds and history
   to the new UID.
4. Master pushes current config/schedule, verifies config_version_hash, runs a self-test
   (DALI scan for a driver device; a metering read for a meter).
5. Master emits a COMMISSION event: old UID, new UID, timestamp, tech ID → permanent audit record.
6. Old UID moves to a decommissioned list (rejoin refused), so a "recovered" unit
   cannot reappear on the network.
```

The rebind-not-reassign discipline comes from [[Device-Identity-Namespaces]].

## Practical field aids

- **QR code on every device carrying its UID.** Costs nothing, saves hours. Scan-plus-GPS at the pole reduces install errors measurably.
- Enforce **serial-to-location mapping, dimming-setting records, and connectivity test records as commissioning deliverables** — this is what prevents warranty disputes later.

## The underestimated task: DALI addressing

Each section needs ~37–53 DALI short addresses mapped to physical poles. **Budget real time for it.** Skip it and "light #27 is out" is useless information. See [[DALI-Bus-Length-Constraint]].

A DALI short address is assigned by a **random-number binary search** and has **no relationship whatsoever to physical position.** The commissioning task is really about building the pole ↔ address table:

```
TERMINATE
INITIALISE          – opens a 15-minute addressing window
RANDOMISE           – each gear generates a 24-bit random address
loop:
  SEARCHADDRH/M/L   – set the search pointer
  COMPARE           – all gear with random_addr <= pointer answer YES
  ... binary search (~24 rounds per device) ...
  PROGRAM SHORT ADDRESS
  VERIFY SHORT ADDRESS
  WITHDRAW          – that gear stops answering COMPARE
TERMINATE
```

Then the physical identification step, which is the labour: **blink-and-confirm** (`DAPC(0)` then `DAPC(254)` on one address, a technician on the street reports which pole flashed) runs 1–2 min per light — **2–4 person-hours for 128 lights, at night, with radio coordination.**

**Avoid that: pre-address on the bench.** Assign short addresses and record the pole ID against the luminaire's barcode *before* it goes up the pole. This converts a night-time street exercise into a daytime bench exercise, and it is the single biggest time saving available in commissioning.

## Replacing a failed driver (existing-driver deployments)

Since the project uses **existing off-the-shelf DALI drivers** (2026-09-13), field replacement is a routine event and the gateway must support it explicitly. A replacement arrives in **factory state**: no short address (status bit 6 set), bit 5 set, `POWER ON LEVEL` 254, `SYSTEM FAILURE LEVEL` 254, `FADE TIME` index 7.

```
1. Replace physically, power up. It will go to full output — expected, not a fault.
2. Broadcast QUERY CONTROL GEAR PRESENT / QUERY MISSING SHORT ADDRESS to find it.
3. Run addressing RESTRICTED TO UNADDRESSED GEAR (INITIALISE with the
   "no short address" argument).
   ⚠ NEVER run a full re-commission on a live bus — it scrambles every address
     and invalidates the whole pole map.
4. PROGRAM SHORT ADDRESS = the SAME address the failed unit had, so the asset
   map stays valid.
5. Re-push ALL configuration — levels, fade, groups, scenes. None of it
   survives the swap.
6. Re-run capability discovery (see [[DALI-Telemetry-And-D4i]]) — the
   replacement may be a different model with different capabilities and a
   different physical minimum.
7. Update the asset record with the new driver GTIN/serial, and reset the
   Part 253 baseline counters for that pole.
   ⚠ The new driver's counters start at zero. If you don't record this, trend
     analysis shows a phantom improvement.
```

Steps 3, 5 and 6 should be **one gateway command**, not a manual sequence. Add a `REPLACE_DRIVER(pole_id)` opcode to the command mailbox — see [[Asynchronous-Command-Job-Pattern]].

## Configuration as code

Keep the full intended configuration for all 128 lights in a version-controlled file on the server, and give the gateway a **"re-assert configuration"** command that pushes it down and verifies by read-back.

After a lightning event, a mains disturbance, or a driver swap, you re-assert rather than reconstruct from memory. **With existing drivers of mixed vintage this matters more, not less** — it is the highest-value operational feature in the gateway's command set.

## Related

- [[Development-Roadmap-And-Gates]]
- [[Gateway-Observability]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://leotek.com/smart-streetlight-commissioning-checklist/ · https://tvilight.com/products/smart-street-light-commissioning-mobile-app/
