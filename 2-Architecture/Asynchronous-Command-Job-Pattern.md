# Asynchronous Command Job Pattern

How writes work when a command can't complete inside a Modbus write's timeout: the server **submits a job and polls its status**. This is the write half of [[Cache-Behind-Synchronous-Wire-Pattern]]. The mailbox, opcodes and job states are specified in `Specs/specs/01-modbus-register-map.md` §8; the result codes in `Specs/specs/02-tier1-protocol.md` §11.5 (v0.1).

## Key points

- The Master answers the job write within 50 ms and reports progress through readback registers and a job table of the last 8 jobs.
- `cmd_seq` is an idempotency key, so the server's own Modbus retries never re-dispatch a command.
- `PARTIAL` is a normal outcome: the per-section fail masks show which devices didn't confirm, and the server retries only those.
- Commands are absolute (`SET_LEVEL(level)`), never relative or toggles.
- **Job TTL is 10 minutes**, and every radio command carries `execute_at` and `not_after`, so a late command is never executed.
- Group, section and site `SET_LEVEL` / `RESUME_SCHEDULE` are **P0**: they jump the queue and go out as broadcasts with `execute_at`.

## Wire flow

```
Server writes:  one FC16 of 0x0100–0x0116:
                cmd_seq, opcode, target, execute_at, params[16], cmd_args_crc, trigger = 0xA5C3
Master replies: within 50 ms; creates a job (or REJECTED with a reason; or exception 0x06 if 8 jobs are open)
Server polls:   job_ack_seq, job_state, job_result, n_targets / n_acked / n_failed, then the job table and fail masks
```

Job states: `IDLE / ACCEPTED / QUEUED / DISPATCHING / PARTIAL / CONFIRMED / FAILED / EXPIRED / REJECTED`. `PARTIAL`, `CONFIRMED`, `FAILED`, `EXPIRED` and `REJECTED` are final.

## Opcodes and targets

The same opcode numbers are used in the mailbox and in radio CMD frames on both tiers:

| Opcode | Command |
|---|---|
| `0x01` | `SET_LEVEL` (level, fade, override TTL ≤ 720 min) |
| `0x02` | `RESUME_SCHEDULE` |
| `0x03` | `IDENTIFY` |
| `0x04` | `QUERY` |
| `0x05` | `SET_TIME` (Master only) |
| `0x06` | `SET_SCHEDULE` |
| `0x07` | `REBOOT` |
| `0x08`–`0x0C` | Commissioning: open window, authorise, revoke, replace device, replace driver |
| `0x0D` | `REASSERT_CONFIG` |
| `0x0E` | `SET_FAILSAFE_LEVELS` |
| `0x0F` | `FORCE_FAILOVER` (tests) |

**Targets** reuse the radio address: a device is its 12-bit address `0x0snn`, a section `0x0sFF`, the whole site `0x0FFF`, the Master `0x0000`, a slave `0x0s00`, and a group `0x8000 | g` (g = 0–15).

**Result codes** are one system-wide table shared by the Modbus job and both radio tiers, e.g. `EXPIRED` (3), `REFUSED_UNSAFE` (4), `TARGET_OFFLINE`, `TIMEOUT`, `AIRTIME_DEFERRED`, `SUPERSEDED`. A newer level command for the same target supersedes one still queued.

## Details that are bugs if omitted

- **Separate command and status register ranges.** Never let the server write a register the Master also writes.
- **A single atomic trigger.** Multi-register writes are only atomic if one FC16 covers the whole block. The block carries a CRC and a final magic trigger value. A server that crashes mid-write then can't launch a half-formed command.
- **`cmd_seq` is the idempotency key.** Resubmitting the same seq returns the existing job's status; it does not re-dispatch. The server never reuses a value within 24 hours.
- **`PARTIAL` is first-class.** With a hundred-plus per-pole devices, "121 confirmed, 7 unknown" is a normal Tuesday.
- **Bounded queue with rejection.** At most 8 open jobs; a ninth gets exception `0x06`. Never accept work you can't deliver.

## Idempotent command rules

1. **Absolute, never relative.** `SET_LEVEL(70%)`, never `DIM_DOWN(10%)`. Relative commands diverge under retry.
2. Every radio command carries a `cmd_id`, `execute_at` and `not_after`, inside a MIC'd frame.
3. A node that has already seen a `cmd_id` **acknowledges it as a duplicate and doesn't execute it again**. The same rule deduplicates P0 copies received on both radios.
4. **The ack carries the resulting state**, not just "OK". For group commands in a cell, confirmation comes from each device's next scheduled report (`level_actual` and `last_cmd_fcnt_lo`), so there is no ACK storm ([[Slave-Cell-Capacity-And-Scheduling]]).
5. **Never make ON/OFF a toggle.** A lost ack plus a retry on a toggle inverts the state permanently.
6. **Frame counters survive reboot:** gateways persist a high-water mark every 64 frames (FRAM), devices every 256 (flash), and jump forward on boot.
7. **A slave may refuse an unsafe command** (e.g. dimming below the schedule while its clock is untrusted) with `REFUSED_UNSAFE`.

## Related

- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Fail-Safe-Lighting-Rule]]
- [[Modbus-Register-Map-Design]]

## Sources

- `Specs/specs/01-modbus-register-map.md` §8, `Specs/specs/02-tier1-protocol.md` §11 (v0.1, 2026-10-20)
- Research synthesis, 2026-09-09. https://industrialmonitordirect.com/blogs/knowledgebase/modbus-race-condition-separating-command-and-status-registers

## History

- 2026-10-14: trigger aligned with the register map's magic value (0xA5C3); confirmation via the device's scheduled report added for per-pole devices.
- 2026-10-21: aligned with spec pack v0.1 (opcodes 0x01–0x0F, 12-bit address targets, shared result-code table, 8-job queue, 10-min TTL, P0 group commands).

Back to [[Index]]
