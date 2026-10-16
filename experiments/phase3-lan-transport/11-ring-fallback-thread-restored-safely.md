# Phase 5 (early attempt #3) — A Low-Frequency Safety-Net Thread Re-added for the Ring (2026-08-28, autonomous round 14 continued)

> **Later note (2026-10-16):** this safety net is what later made pipelining safe. Phase 4
> removed the unconditional submit-time push that this note keeps (`phase4-wan-aws/04-*.md`,
> 435 s → 236 s), relying on this thread to deliver purely asynchronous command sequences.

## The change made

After the finding in note 10: the ring's synchronization depended ENTIRELY on the active
push/pull hooks (the background thread had been disabled completely, note 07 Bug #8). This
created an architecture with no delivery guarantee (the hang in note 10 proved it). As the
fix, the background thread was re-added for the ring as a safety net **at a low frequency
(5ms, 25x less often than before)** (`vn_renderer_vtest.c`, `vtest_shmem_sync_thread_func`).

## Results

- **Correctness:** 3/3 clean runs at 0ms delay (`driverName = venus`).
- **40ms artificial delay (tc netem, isolated namespace):** 86.0s — the same as the earlier 81-85s range (within statistical noise); **as expected**, no change in speed (the push is still unconditional; this change only adds a safety net, it does not reduce the bottleneck).

## Assessment

This turned out not to be a SPEED optimization but a **correctness/robustness fix** — it
closes the dangerous gap from note 10 (the risk of async commands never being delivered) and
does not touch performance. Current state: the architecture is both correct and (relatively)
safe, but still not optimized (289 commands × mandatory round trips).

## Overall status summary (end of this round)

Of the 3 optimization ideas tried this round:
1. **Throttled active pull** (note 09) → no measurable benefit, harmless, kept.
2. **Idle-conditional push** (note 10) → broke correctness, COMPLETELY reverted.
3. **Low-frequency ring fallback thread** (this note) → a correctness/robustness improvement, no speed gain, kept.

**Conclusion:** phase 3's proof of correctness stands solid and has now become more robust.
A real speed gain needs the "real pipelining" described in note 09 (accumulating async
commands and sending them in bulk) — recorded as a bigger engineering task, beyond the scope
of this session, that requires touching the upper layers of `vn_ring_submit_command`.

## Recommendation for the next steps

The AWS quota answer will not come before tomorrow (the user said so). The results this
session has produced up to this point are already very valuable (phase 3 fully proven + 3
optimization attempts honestly documented, one of them a permanent improvement). This is a
reasonable stopping point for the autonomous loop — the real pipelining redesign is a big and
risky next step, and starting it autonomously without the user's informed approval may not be
healthy.
