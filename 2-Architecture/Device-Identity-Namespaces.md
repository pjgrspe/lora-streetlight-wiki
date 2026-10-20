# Device Identity Namespaces

Three separate identity namespaces exist in this system: hardware UID, logical short address, and asset/location ID. Conflating any two is a design error that surfaces later as ghost devices and hardware that can't be replaced. Addressing is specified in `Specs/specs/00-common-frame-and-conventions.md` §2.3 and §3 (v0.1).

## Key points

- Radio frames carry only the **12-bit address** `GwID:4 | NodeID:8`, never the UID (airtime). The same 12-bit address is the device reference in the Modbus map and the command target.
- The master's registry is the **only** place `UID ↔ short_addr ↔ asset_id` is bound, and it's mirrored to the server.
- Device swap = **rebind the new UID to the same short address and asset ID**.
- Addresses are section-scoped. GwID 1–14 (one per section), NodeID 1–63 in v1 (60 used at most), 64–253 reserved for growth.
- **The Slave always gives the same NodeID back to the same UID.**
- The registry lives in a transactional, power-loss-safe store.

## The three namespaces

| Namespace | Example | Assigned by | Immutable? | Used for |
|---|---|---|---|---|
| **UID** (hardware) | 96-bit (12-byte) STM32 unique device ID | Factory / bench | Yes | Provisioning, key derivation, warranty, anti-clone |
| **Address** (12-bit) | `(GwID << 8) \| NodeID` within the site's 8-bit NetID, e.g. `0x311` = section 3, node 17 ([[Custom-Protocol-Frame-Design]]) | The Slave, at join | No, reassignable | Every radio frame |
| **Asset / location ID** | `PH-SITE1-SEC2-POLE-017` | Project / GIS | Yes for the *pole*, not the device | Maps, work orders, field technicians |

## Rules

- **Radio frames carry the short address only.** A 6–8 byte UID in every frame is airtime you can't afford ([[LoRa-Airtime-Budget]]).
- **Device swap = rebind.** The new UID takes over the old short address, asset ID, config and history. This turns field replacement into a 10-minute job instead of a reconfiguration project ([[Commissioning-And-Field-Swap]]).
- **Short addresses are section-scoped:** the GwID identifies the section's Slave, so an address alone tells you where a device is.
- **Never allocate a fresh address to a known UID.** A node that forgot its address must rejoin and get the *same* one back; otherwise the registry accumulates ghosts.
- **Size the space generously.** Up to 60 devices per Slave, an unknown number of sections, and future growth. GwID 1–14 plus NodeID 1–63 (with 64–253 reserved) means growth never forces a renumbering.
- **Special addresses:** NodeID 0 = the gateway itself, 254 = an unjoined device (in `JOIN_REQ` only), 255 = all devices in the section. GwID 0 = the Master, 15 = all gateways. `0xFFF` = the whole site.

## Registry fields to persist

Stored in non-volatile storage, not just RAM:

```
uid, short_addr, type, fw_version, hw_rev, parent_addr, section, asset_id,
install_date, key_index, frame_counters (tx/rx), commissioned_state,
home_channel, slot_index, expected_report_interval, alarm_thresholds,
last_seen, cumulative counters, config_version_hash
```

`home_channel` and `slot_index` are the device's cell assignment ([[Slave-Cell-Capacity-And-Scheduling]]). The Slave keeps its own `UID → {NodeID, slot, home channel}` table, plus session keys and frame counters, in FRAM, so a Slave reboot doesn't force every device to rejoin.

## Swap in practice

The server sends `REPLACE_DEVICE` with the old address and the new UID. The new device joins, the Slave gives it the old NodeID and slot, and the old UID is revoked so it can never rejoin. The server's `device_assignment` table closes the old row and opens a new one with the same address and pole, so the pole's history survives. Specified in `Specs/specs/03-tier2-device-protocol.md` §12.2 and `Specs/specs/07-server-data-model.md` §4.1 (v0.1).

Store the registry in a **transactional, wear-aware structure**: a journal plus two-copy commit with CRC ([[Persistence-And-Storage]]). A registry corrupted by a brownout orphans the whole network, a genuine bricking risk in a Philippine street-lighting cabinet. **Mirror it to the Local Monitoring Server** so it can be restored without re-commissioning the site.

## Related

- [[Modbus-Register-Map-Design]]
- [[Custom-Protocol-Security]]
- [[Commissioning-And-Field-Swap]]

## Sources

- `Specs/specs/00-common-frame-and-conventions.md` §2.3, §3; `Specs/specs/03-tier2-device-protocol.md` §4.4, §12.2 (v0.1)
- Research synthesis, 2026-09-09.

## History

- 2026-10-14: address scheme aligned with the frame design's GwID/NodeID, sized for per-pole devices and an unknown section count; cell-assignment fields added.
- 2026-10-21: aligned with spec pack v0.1 (12-bit address, NodeID 1–63 in v1, UID = 96-bit STM32 ID, same NodeID back to the same UID, swap rebind via REPLACE_DEVICE).

Back to [[Index]]
