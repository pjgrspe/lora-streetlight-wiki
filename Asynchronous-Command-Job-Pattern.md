# Asynchronous Command Job Pattern

Writes are the hard half of [[Cache-Behind-Synchronous-Wire-Pattern]]. A command cannot complete inside a Modbus write's timeout, so the server submits a **job** and polls its status.

## Wire flow

```
Server writes:  CMD_TARGET, CMD_OPCODE, CMD_ARG[0..n], CMD_SEQ, then CMD_TRIGGER=1
Master returns: JOB_ID (echoing CMD_SEQ), JOB_STATE=ACCEPTED
Server polls:   JOB_STATE, JOB_PROGRESS (n_acked/n_targets), JOB_RESULT, JOB_FAIL_MASK
```

`JOB_STATE` enum: `IDLE / ACCEPTED / QUEUED / DISPATCHING / PARTIAL / CONFIRMED / FAILED / EXPIRED / REJECTED`

## Details that are bugs if omitted

- **Separate command and status register ranges.** Never let the server write a register the master also writes. The convention exists to kill that race.
- **Single atomic trigger.** Multi-register Modbus writes are not atomic unless FC16 covers the whole block. Use FC16 for the argument block, require a final trigger write, and validate a checksum over the arguments before accepting.
- **`CMD_SEQ` is the idempotency key.** A resubmitted identical seq returns the existing job's status — it does not re-dispatch. This is what protects you from the server's *own* Modbus retries, which will duplicate writes.
- **`PARTIAL` is a first-class outcome**, not an error. With 128 luminaires, "121 confirmed, 7 unknown" is a normal Tuesday. Expose `JOB_FAIL_MASK` so the server retries only the stragglers.
- **Job TTL.** Expire jobs (~10 min) and carry an `execute_not_after` in the wire frame. A "turn on" that lands 3 hours late is a hazard, not a success.
- **Bounded queue with rejection.** Queue full → `REJECTED` immediately. Never accept work you cannot deliver.

## Idempotent command rules

1. **Absolute, never relative.** `SET_LEVEL(70%)`, never `DIM_DOWN(10%)`. Relative commands are non-idempotent by construction and diverge under retry.
2. Every frame carries `{node_addr, cmd_seq, opcode, args, execute_at, not_after, MIC}`.
3. Receiver keeps `last_seq` plus a sliding window; a seq at or below `last_seq` is **acked but not executed**.
4. **The ack carries resulting state, not just "OK"** — `ack{seq, actual_level, status}` removes the need for a separate read, halving airtime and killing the "did it work?" ambiguity.
5. **Never make ON/OFF a toggle.** A lost ack plus a retry on a toggle inverts state permanently.
6. Frame counters must survive reboot (persist every ~64 counts, jump forward on boot), or a power cycle re-opens the replay window.

## Related

- [[Popcorn-Effect-And-Scheduled-Execution]]
- [[Fail-Safe-Lighting-Rule]]
- [[Modbus-Register-Map-Design]]
- [[Index]]

## Sources

- Research synthesis, 2026-09-09. https://industrialmonitordirect.com/blogs/knowledgebase/modbus-race-condition-separating-command-and-status-registers
