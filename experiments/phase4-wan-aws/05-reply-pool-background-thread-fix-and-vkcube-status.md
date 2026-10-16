# Phase 5 — Reply-Pool Background Thread Fix + Where vkcube Stands (2026-08-29)

## A second bottleneck found

In the vkcube test, the same line was seen repeated hundreds of times in the host log (`VIRGL_LOG_LEVEL=debug`):
```
[DEBUG] server transfer_get on reply-pool res: handle=2 offset=0 size=1048576 ...
```

Root cause: the reply pool's background synchronization thread
(`vtest_shmem_sync_thread_func`, `vn_renderer_vtest.c`) had been doing **a blind GET of the
whole 1MB every 2ms** since phase 3 — next to the active `sync_now` hook (which has the
`hint_size`/delta optimization added in phase 4), this background thread had never been
optimized. Trying to pull 1MB every 2ms over the real internet created continuous, largely
unnecessary traffic + mutex contention (the ring's own background thread had already been
moved to 5ms, but the reply-pool branch had been forgotten).

## Fix

The reply-pool background thread's period was raised from 2ms to **100ms**
(`vn_renderer_vtest.c`, `vtest_shmem_sync_thread_func`). The active `sync_now` hook already
does the real work (the delta pull); it is enough for this background thread to be a
low-frequency safety net for rare "forgotten" cases.

## Result

In the vkcube test: before the fix, 8 synchronous calls completed in 90 seconds (~11.25s per
call); after the fix, **15 synchronous calls in 60 seconds** (~4s per call) — **a ~2.8x
speed-up**.

## Where vkcube stands now

After ~150 seconds the process is stuck at the 16th synchronous call (it stayed at 15); ring
synchronizations continue at around ~950ms-1.2s but do not complete — it has entered
`vn_relax`'s exponential backoff. **No crash**, just very slow. This is probably part of the
device-memory/image creation chain (`vtest_bo_create_from_device_memory`,
`vtest_vcmd_resource_create_blob`) — it uses **an RPC mechanism completely separate** from
the Venus ring's command stream (a direct, synchronous `vtest_write`+`vtest_read`), so it gets
no benefit at all from this session's pipelining fix (which is specific to the ring).

The window has not opened yet — completing the swapchain creation chain is still a long way off.

## Summary of this round

In this session, after moving from `vulkaninfo` to trying `vkcube`, the problems found/fixed were, in order:
1. The missing device-memory transport (crash → made to work, note 03).
2. The unconditional push at ring submit time (a pipelining obstacle → removed, a 46% speed-up, note 04).
3. The overly aggressive (2ms, full 1MB) synchronization of the reply-pool background thread (→ lowered to 100ms, a ~2.8x speed-up, this note).

All three are real, measurable gains. The remaining open question: why the device-memory /
blob-create RPC chain itself is so slow — this needs a separate diagnostic round (probably a
similar "aggressive background thread" or "unnecessarily large transfer" pattern, but not yet
verified).
