# Asynchronous Command Job Pattern

How writes work when a command can't complete inside a Modbus write's timeout: the server **submits a job and polls its status**. This is the write half of [[Cache-Behind-Synchronous-Wire-Pattern]].

## Key points

- The master accepts a job immediately (under 50 ms) and reports progress through status registers.
- `CMD_SEQ` is an idempotency key, so the server's own Modbus retries never re-dispatch a command.
- `PARTIAL` is a normal outcome: report it with a fail mask and retry only the stragglers.
- Commands are absolute (`SET_LEVEL(70%)`), never relative or toggles.
- Jobs expire, so a late command is never executed hours after the fact.

## Wire flow

```
Server writes:  CMD_TARGET, CMD_OPCODE, CMD_ARG[0..n], CMD_SEQ, then CMD_TRIGGER = 0xA5C3
Master returns: JOB_ID (echoing CMD_SEQ), JOB_STATE = ACCEPTED
Server polls:   JOB_STATE, JOB_PROGRESS (n_acked/n_targets), JOB_RESULT, JOB_FAIL_MASK
```

`JOB_STATE` values: `IDLE / ACCEPTED / QUEUED / DISPATCHING / PARTIAL / CONFIRMED / FAILED / EXPIRED / REJECTED`

The register layout of the mailbox is in [[Modbus-Register-Map-Design]].

## Details that are bugs if omitted

- **Separate command and status register ranges.** Never let the server write a register the master also writes.
- **A single atomic trigger.** Multi-register writes are only atomic if one FC16 covers the whole block. Write the argument block with FC16, require a final magic trigger value, and validate a checksum over the arguments before accepting. A server that crashes mid-write then can't launch a half-formed command.
- **`CMD_SEQ` is the idempotency key.** Resubmitting the same seq returns the existing job's status; it does not re-dispatch.
- **`PARTIAL` is first-class.** With a hundred-plus per-pole devices, "121 confirmed, 7 unknown" is a normal Tuesday. Expose `JOB_FAIL_MASK` so the server retries only the stragglers.
- **Job TTL.** Expire jobs after ~10 min and carry `execute_not_after` in the radio frame. A "turn on" that lands 3 hours late is a hazard, not a success.
- **Bounded queue with rejection.** Queue full → `REJECTED` immediately. Never accept work you can't deliver.

## Idempotent command rules

1. **Absolute, never relative.** `SET_LEVEL(70%)`, never `DIM_DOWN(10%)`. Relative commands diverge under retry.
2. Every frame carries `{node_addr, cmd_seq, opcode, args, execute_at, not_after, MIC}`.
3. The receiver keeps `last_seq` plus a sliding window. A seq at or below `last_seq` is **acked but not executed**.
4. **The ack carries the resulting state**, not just "OK": `ack{seq, actual_level, status}`. That removes a separate read, halves airtime, and ends the "did it work?" ambiguity. In the device cell, the device's next scheduled report carries `level_actual` and serves as the confirmation ([[Slave-Cell-Capacity-And-Scheduling]]).
5. **Never make ON/OFF a toggle.** A lost ack plus a retry on a toggle inverts the state permanently.
6. **Frame counters survive reboot:** persist every ~64 counts and jump forward on boot, or a power cycle re-opens the replay window. (Devices on the Blue Pill write the high-water mark less often, e.g. every 256 frames, to spare flash; see [[STM32-Part-Selection]].)

## Related

- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Fail-Safe-Lighting-Rule]]
- [[Modbus-Register-Map-Design]]

## Sources

- Research synthesis, 2026-09-09. https://industrialmonitordirect.com/blogs/knowledgebase/modbus-race-condition-separating-command-and-status-registers

## History

- 2026-10-14: trigger aligned with the register map's magic value (0xA5C3); confirmation via the device's scheduled report added for per-pole devices.

Back to [[Index]]
