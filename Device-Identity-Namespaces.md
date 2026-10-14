# Device Identity Namespaces

Three separate identity namespaces exist in this system. Conflating any two of them is a design error that surfaces later as ghost devices and unreplaceable hardware.

| Namespace | Example | Assigned by | Immutable? | Used for |
|---|---|---|---|---|
| **UID** (hardware) | 96-bit STM32 UID, or a 64-bit EUI you allocate | Factory / bench | Yes | Provisioning, key derivation, warranty, anti-clone |
| **Logical short address** | 1 byte: `0x01..0x03` slaves, `0x11..0x36` endpoints | Master, at commissioning | No — reassignable | Every radio frame |
| **Asset / location ID** | `PH-SITE1-PHASE_A-POLE-017` | Project / GIS | Yes for the *pole*, not the device | Maps, work orders, field techs |

## Rules

- **Radio frames carry the short address only.** A 6–8 byte UID in every frame is airtime you cannot afford — see [[LoRa-Airtime-Budget]].
- The master's registry is the **only** place `UID ↔ short_addr ↔ asset_id` is bound.
- **Device swap = rebind the UID to the same short address and asset ID.** This single choice is what turns field replacement into a 5-minute job instead of a reconfiguration project — see [[Commissioning-And-Field-Swap]].
- Short addresses are **section-scoped** (`0x1n` = section A endpoints, etc.) so an address alone tells you where a device is.
- **Never allocate a fresh address to a known UID.** A node that forgot its address must rejoin and get the *same* one back; otherwise the registry accumulates ghosts.
- Size the address space at **12–16 bits**, well beyond 130, so growth never forces a renumbering.

## Registry fields to persist

Stored in external flash/EEPROM, not just RAM:

```
uid, short_addr, type, fw_version, hw_rev, parent_addr, section, asset_id,
install_date, key_index, frame_counters (tx/rx), commissioned_state,
expected_report_interval, alarm_thresholds, dali_child_count, last_seen,
cumulative counters, config_version_hash
```

Store it in a **transactional, wear-aware structure** — journal plus two-copy commit with CRC. A registry corrupted by a brownout orphans the entire network, which is a genuine bricking risk in a Philippine street-lighting cabinet.

Mirror the registry to the Local Monitoring Server so it is restorable without re-commissioning the site.

## Related

- [[Modbus-Register-Map-Design]]
- [[Custom-Protocol-Security]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09.
