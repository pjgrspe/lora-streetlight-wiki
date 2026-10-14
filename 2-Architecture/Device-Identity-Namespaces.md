# Device Identity Namespaces

Three separate identity namespaces exist in this system: hardware UID, logical short address, and asset/location ID. Conflating any two is a design error that surfaces later as ghost devices and hardware that can't be replaced.

## Key points

- Radio frames carry only the **short address**, never the UID (airtime).
- The master's registry is the **only** place `UID ↔ short_addr ↔ asset_id` is bound, and it's mirrored to the server.
- Device swap = **rebind the new UID to the same short address and asset ID**.
- Addresses are section-scoped and sized well beyond today's count, because the number of sections and devices is still TBC.
- The registry lives in a transactional, power-loss-safe store.

## The three namespaces

| Namespace | Example | Assigned by | Immutable? | Used for |
|---|---|---|---|---|
| **UID** (hardware) | 96-bit STM32 UID, or a 64-bit EUI you allocate | Factory / bench | Yes | Provisioning, key derivation, warranty, anti-clone |
| **Logical short address** | `[GwID : 4][NodeID : 8]` within the site's NetID ([[Custom-Protocol-Frame-Design]]) | Master/Slave, at commissioning | No, reassignable | Every radio frame |
| **Asset / location ID** | `PH-SITE1-SEC2-POLE-017` | Project / GIS | Yes for the *pole*, not the device | Maps, work orders, field technicians |

## Rules

- **Radio frames carry the short address only.** A 6–8 byte UID in every frame is airtime you can't afford ([[LoRa-Airtime-Budget]]).
- **Device swap = rebind.** The new UID takes over the old short address, asset ID, config and history. This turns field replacement into a 10-minute job instead of a reconfiguration project ([[Commissioning-And-Field-Swap]]).
- **Short addresses are section-scoped:** the GwID identifies the section's Slave, so an address alone tells you where a device is.
- **Never allocate a fresh address to a known UID.** A node that forgot its address must rejoin and get the *same* one back; otherwise the registry accumulates ghosts.
- **Size the space generously.** Up to 60 devices per Slave today, an unknown number of sections, and future growth. GwID 1–14 and NodeID 1–254 leave ample headroom, so growth never forces a renumbering.

## Registry fields to persist

Stored in non-volatile storage, not just RAM:

```
uid, short_addr, type, fw_version, hw_rev, parent_addr, section, asset_id,
install_date, key_index, frame_counters (tx/rx), commissioned_state,
home_channel, slot_index, expected_report_interval, alarm_thresholds,
last_seen, cumulative counters, config_version_hash
```

`home_channel` and `slot_index` are the device's cell assignment ([[Slave-Cell-Capacity-And-Scheduling]]). The Slave persists them too, so a Slave reboot doesn't force every device to re-join.

Store the registry in a **transactional, wear-aware structure**: a journal plus two-copy commit with CRC ([[Persistence-And-Storage]]). A registry corrupted by a brownout orphans the whole network, a genuine bricking risk in a Philippine street-lighting cabinet. **Mirror it to the Local Monitoring Server** so it can be restored without re-commissioning the site.

## Related

- [[Modbus-Register-Map-Design]]
- [[Custom-Protocol-Security]]
- [[Commissioning-And-Field-Swap]]

## Sources

- Research synthesis, 2026-09-09.

## History

- 2026-10-14: address scheme aligned with the frame design's GwID/NodeID, sized for per-pole devices and an unknown section count; cell-assignment fields added.

Back to [[Index]]
