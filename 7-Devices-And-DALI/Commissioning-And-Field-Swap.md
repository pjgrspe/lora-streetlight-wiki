# Commissioning And Field Swap

How each per-pole device and section meter is identified, located, enrolled and configured, and how a failed device or LED driver is replaced in the field. With one device per pole, commissioning is mostly about **binding each device's UID to its pole**, not night-time DALI addressing.

## Key points

- **Hybrid join:** root key + UID provisioned on the bench; short address and session keys assigned over the air only during a time-boxed, authorised commissioning window.
- **Pole mapping = device UID ↔ pole ID**, captured at install by scanning a QR code with GPS. This replaces bus-wide DALI addressing and blink-and-confirm.
- **Field swap target:** one technician, no laptop, under 10 minutes, no server reconfiguration. The gateway **rebinds** the old pole's identity to the new UID.
- **Driver replacement is a command** (`REPLACE_DRIVER`, opcode 0x0C) that triggers the device-side DALI routine and re-pushes configuration. **Device swap** is `REPLACE_DEVICE` (0x0B).
- **The same UID always gets the same NodeID back**, and a device that reboots with a stored slot rejoins **in its own slot**, so a mains restore causes no join storm. Specified in `Specs/specs/03-tier2-device-protocol.md` §4–§5 and §12 (v0.1).
- **Survey the radio links before freezing anything**, with the real radios at the real heights.

## Who does what

| Task | Device (device developer) | Slave/Master gateway | Server |
|---|---|---|---|
| Bench provisioning of UID + root key | Firmware support | — | Provisioning record |
| Join and slot assignment | Device-role library | Whitelist check, slot assignment ([[Slave-Cell-Capacity-And-Scheduling]]) | Authorises UIDs per section |
| In-pole DALI addressing and config | **Runs it** | Sends the command and config | Holds intended config |
| Capability discovery | **Runs it** ([[DALI-Telemetry-And-D4i]]) | Stores the bitmap | Displays it |
| UID ↔ pole binding, swap rebind | — | **Rebinds** registry entry | Starts the swap; keeps the audit record |

## Join model: hybrid

| Model | Pros | Cons |
|---|---|---|
| Fully pre-provisioned (keys + address burned on the bench) | Simple, no join attack window | Every swap needs a bench trip; bookkeeping errors |
| Open over-the-air pairing | Trivial swap | Anyone with a cheap radio can join |
| **Hybrid (chosen)** | **Root key + UID on the bench; short address and session keys over the air only in a time-boxed, physically authorised window** | Some protocol work |

Key derivation: `DevKey = AES-CMAC(SiteKey, UID)`, so per-device keys are never stored in a table. The site key lives only in the gateways (Master and Slaves), never on a device. See [[Custom-Protocol-Security]] and spec 00 §6.

**How a join works (spec 03):**

1. The server authorises the UID for a section (`AUTHORISE_DEVICE`, 0x09). The Master pushes a `WHITELIST` entry to that section's Slave.
2. The Slave opens its commissioning window (`OPEN_COMMISSIONING`, 0x08), shown as a flag in its cell beacon. A never-joined UID is accepted only while it is open.
3. The device scans the 12 cell channels for a beacon, then sends `JOIN_REQ` (UID, nonce, type, firmware) in the contention window with random backoff.
4. The Slave answers `JOIN_ACC` with the NodeID, home channel, slot, cycle parameters and cell key. ⚠ A first join can take up to 24 min because of the channel scan (spec 03 O-03-3).

## Radio site survey (before enclosures and channel plan are frozen)

1. **Use the actual radios, heights and antennas.** Real urban range is usually 2–5 km against a theoretical 15 km.
2. Record RSSI/SNR/PER at every slave location and representative poles, at the working and fallback SF, and the **noise floor with all transmitters off**.
3. **Soak each link 24 h+**, including at least one rain measurement.
4. Accept **≥15 dB margin at the working SF**; aim for ~25 dB in vegetated terrain.
5. Document antenna type, height, polarisation, cable length/loss and orientation photos. **Connector water ingress is the most common slow failure in tropical installs.**
6. Scan the 915–918 MHz band for existing occupancy before fixing channels ([[Philippines-Regulatory-Context]]).

Details in [[Link-Budget-And-Site-Survey]].

## Installing a pole

1. Device arrives bench-provisioned (UID, root key, QR label).
2. Technician mounts it and **scans the QR + GPS** against the pole ID in the server.
3. Server authorises that UID for the section; the device joins in the commissioning window and gets its slot.
4. Device runs in-pole DALI setup and capability discovery; gateway pushes config (levels, fade, schedule) and checks `config_version_hash`.
5. A COMMISSION event (UID, pole, timestamp, technician) is logged permanently.

**In-pole DALI addressing is trivial** with one driver per pole: broadcast or a fixed short address. The 128-light random-address binary search and night-time blink-and-confirm (2–4 person-hours) are no longer needed. If a pole carries several luminaires, the device runs the standard procedure on its own short link:

```
TERMINATE → INITIALISE (15-min window) → RANDOMISE
loop: SEARCHADDRH/M/L → COMPARE (binary search) → PROGRAM SHORT ADDRESS → VERIFY → WITHDRAW
TERMINATE
```

## Swapping a per-pole device

```
1. Server/UI: REPLACE_DEVICE(old address, new UID) for pole PH-S1-P027
   → Master sends a WHITELIST entry with the replacement flag; old UID decommissioned
   → Slave opens a 5-minute commissioning window for that NodeID only
2. Technician installs the new device and scans its QR.
3. New device joins normally. The Slave gives it the OLD device's NodeID,
   slot and home channel (rebind, not reassign).
4. Slave pushes SET_SCHEDULE and REASSERT_CONFIG; the device runs DALI setup
   and capability discovery. Slave raises DEVICE_SWAPPED.
5. Server records old UID, new UID, timestamp, technician → audit record.
6. Joins from the old UID are refused from then on, so a recovered or
   stolen unit cannot reappear.
```

Rebind-not-reassign comes from [[Device-Identity-Namespaces]].

## Replacing a failed LED driver

With existing off-the-shelf drivers, driver replacement is routine. A new driver arrives in **factory state**: no short address (status bit 6 set), bit 5 set, `POWER ON LEVEL` 254, `SYSTEM FAILURE LEVEL` 254, `FADE TIME` index 7. It will go to full output on power-up; that's expected.

One command, `REPLACE_DRIVER` (0x0C) to the pole's device, triggers it to:

1. Find the new gear (`QUERY CONTROL GEAR PRESENT` / `QUERY MISSING SHORT ADDRESS`).
2. Address **only unaddressed gear** (INITIALISE with the "no short address" argument). ⚠ **Never** run a full re-commission on a link with other addressed gear.
3. Program the **same short address** the failed unit had.
4. Re-push **all** configuration (levels, fade, groups, scenes); none survives a swap. The fail-safe levels make this a safety step, not a convenience.
5. Re-run capability discovery; the replacement may be a different model with a different physical minimum.
6. Report the new driver GTIN/serial; the server updates the asset record and **resets Part 253 baseline counters** for that pole (otherwise trend analysis shows a phantom improvement).

Steps 1–5 run in device firmware; the device answers with a CMD_ACK and a `DALI_GEAR_REPLACED` event, and the gateway verifies the result (spec 03 §12.3).

## Configuration as code

The server keeps the full intended configuration for every device (spec 07 `config_intent`), and the system supports **`REASSERT_CONFIG`** (0x0D): push it down, have the device rewrite its DALI settings and verify by read-back. A mismatch sets the device's `config_mismatch` status bit. After lightning, a mains disturbance or a swap, re-assert rather than reconstruct from memory. With mixed-vintage drivers this is the highest-value operational command.

## Practical field aids

- **QR code on every device with its UID.** Scan + GPS at the pole cuts install errors.
- Commissioning deliverables: serial-to-pole mapping, dimming settings, connectivity test record per pole. These prevent warranty disputes later.

## Related

- [[Device-Identity-Namespaces]]
- [[Custom-Protocol-Security]]
- [[Link-Budget-And-Site-Survey]]
- [[DALI-Telemetry-And-D4i]]
- [[Development-Roadmap-And-Gates]]
- [[Gateway-Observability]]

## Sources

- `Specs/specs/03-tier2-device-protocol.md` §4, §5, §12; `Specs/specs/02-tier1-protocol.md` (WHITELIST); `Specs/specs/07-server-data-model.md` (v0.1).
- Research synthesis, 2026-09-09. https://leotek.com/smart-streetlight-commissioning-checklist/ · https://tvilight.com/products/smart-street-light-commissioning-mobile-app/

## History

- 2026-09-09: written for section-wide DALI buses, with bench pre-addressing and blink-and-confirm for 128 lights.
- 2026-09-13: driver-replacement procedure added for existing off-the-shelf drivers.
- 2026-10-14: per-pole devices; commissioning became UID ↔ pole binding, and DALI steps moved into device firmware.
- 2026-10-21: aligned with spec pack v0.1 (join and swap flows per 03, opcodes REPLACE_DEVICE/REPLACE_DRIVER/REASSERT_CONFIG, same NodeID per UID, rejoin in own slot, site key in gateways only).

Back to [[Index]]
