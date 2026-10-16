# Phase 5 — Real Pipelining: 435s → 236s (a 46% Speed-up) (2026-08-29)

## The idea

In `vn_ring_submit_internal` (since phase 3), after EVERY submit (sync or async alike) we
pushed the ring to the host **unconditionally and immediately**. This made even async
commands (fire-and-forget, `vn_async_vkFoo`, which never call `vn_ring_wait_seqno`) pay their
own network round trip.

Notes 09/10 identified exactly this, but the attempted fix at the time (removing the
unconditional push) had **broken correctness** — because by then the ring's background
synchronization thread had been removed completely (the Bug #8 fix), so there was NO delivery
path at all for async data left without a push.

## Why it is safe now

Two things changed in this session:
1. **The background thread came back** (note 11) — at a low frequency (5ms), a safety net with bounded latency (at worst ~5ms) for purely async command sequences.
2. **The race-condition fix** (note 02) — `vn_ring_write_buffer` is now under a lock, so the background thread can no longer read half-written data.

Thanks to these two safety nets, removing the submit-time push **entirely** is now safe: the
data reaches the host sooner or later, either (a) through the next SYNCHRONOUS call's own
`vn_ring_wait_seqno` loop (which already always triggers a push+pull cycle on its first
iteration), or (b) through the background thread for purely async sequences — but no longer
for EVERY SINGLE command, ACCUMULATED instead.

## The change

`vn_ring.c`, `vn_ring_submit_internal()`: the unconditional `vn_renderer_shmem_sync_now(...)`
call after `store_tail` was **removed entirely**. Everything else (lock/unlock, store_tail)
stayed the same.

## Result

In a clean AWS test (the same server, the same real internet connection):

| | Before (unconditional push) | After (pipelining) |
|---|---|---|
| Total time | 435s | **236s** |
| Ring sync calls | 578 | **353** |
| Crash | None | None |
| Result | `driverName=venus`, NVIDIA A10G | Same |

**A 46% speed-up, 39% fewer ring round trips, correctness preserved.** This shows that the
pipelining work the plan had marked as "big and risky, a separate phase 5 task" was possible
with a much smaller change than predicted (removing a single function call) — because the
existing `wait_seqno` mechanism was already acting as a natural "batch point" for sync calls;
we just were NOT USING it.

## Next step

To be tested on vkcube (real rendering, a workload with many consecutive async commands) —
since it has a much higher ratio of async commands than vulkaninfo, the effect of pipelining
there will probably be even bigger.
