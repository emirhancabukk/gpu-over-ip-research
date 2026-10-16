# Phase 3 — Reply-Pool Freshness Hook Added (Works); A Separate, Intermittent "CS Error" Remains (2026-08-28, autonomous round 10)

## Done this round: a synchronous reply-pool "sync_now" hook

The concrete task left by the previous round (note 04) was implemented: instead of blind
periodic polling, a synchronous GET for the pending reply-pool resource is triggered at
EXACTLY the point where `vn_ring_wait_seqno()` (`vn_ring.c`) returns successfully.

**Why this is the right place:** the Venus protocol's own ordering guarantee — the host does
not advance the ring seqno until it has finished writing a command's reply. So the moment
`vn_ring_get_seqno_status()` returns true and `wait_seqno` exits successfully, the host's
reply (if any) is DEFINITELY ready. It was not guaranteed that the background poll thread
had CAUGHT this (the actual gap found by note 04); this round's fix closes that gap.

**Implementation (3 files):**
- `vn_renderer.h`: added an optional `sync_now` field to `struct vn_renderer_shmem_ops` + a `vn_renderer_shmem_sync_now()` inline helper (a no-op if the renderer does not support it).
- `vn_renderer_vtest.c`: added `vtest_shmem_sync_now()` — it does a synchronous `vtest_vcmd_transfer_get2` only for `sync_active` resources (those with a background thread in TCP mode); hooked up to `shmem_ops.sync_now`.
- `vn_ring.c`: added a call to `vn_renderer_shmem_sync_now(ring->instance->renderer, ring->instance->reply_shmem_pool.shmem)` at the success return point of `vn_ring_wait_seqno()`.

## Result: partial but real improvement

This hook **completely solved the reply freshness problem of the first call
(vkEnumerateInstanceVersion)** — in **all** of 5 consecutive runs, `command_type=137`
(exactly matching the expected value) was decoded correctly, a clear improvement over the
previous round's ~50% success rate.

However, the process now gets stuck/becomes very slow **at a later point**
(`vkEnumeratePhysicalDevices`, the next synchronous Venus call). A `SIGABRT` was sent to a
live run and the core dump was analysed with `coredumpctl` — this is **not a deadlock**; it
was still legitimately waiting inside `vn_ring_wait_seqno` → `os_time_sleep`. So progress is
being made, but much more slowly than expected.

## Separate problem newly observed: the "CS error" still occurs in some runs

In the next test round (a bounded 60-second test), the `vkCreateInstance resulted in CS
error` / `ring_submit_cmd: vn_dispatch_command failed` error, **thought to be "completely
fixed" in the previous round**, was observed **again** — this time the reply pool never got
past zero (the host never managed to process the command successfully, so no reply was
ever written).

This shows that the "PUT the buffer before the tail" fix (note 03, Bug #6) **reduced the
problem but did not eliminate it completely** — there is still an intermittent
(non-deterministic) data-visibility or timing problem between the host's ring consumer
thread and the main dispatch thread. Possible causes (not verified, hypotheses for the next
round):
- The host's ring consumer thread (`vkr_ring.c`) may check for tail changes periodically (with cnd_wait/timeout), and between OUR consecutive PUTs (buffer, then tail) the host may have ALREADY started reading with another, older tail value at THAT moment (that is, the problem may not be the order; MULTIPLE OVERLAPPING synchronization cycles/repeated PUTs may be confusing the host — on EVERY cycle we re-send the WHOLE 128KB buffer, not only the newly written part).
- Possibly: ring wrap-around behaviour is not handled properly.

## Concrete Task for the Next Round

1. Add debug prints on the server side to understand how/when the ring consumer thread in `vkr_ring.c` reads the tail, and EXACTLY which bytes the host sees at the moment of the decode error (dump the first few hundred bytes of the ring buffer at the moment of the decode error).
2. Investigate why `vkEnumeratePhysicalDevices` is so slow — probably it is not enough for the sync_now hook to run ONLY on `wait_seqno`'s SUCCESS path; the synchronization speed of the ring ITSELF (head/status) still depends on the background thread's 200µs loop. For this, a similar "event-triggered" approach for the ring synchronization (e.g. trigger a synchronous ring sync immediately when a new command is sent, instead of relying only on the background thread) can be tried.
3. If no progress can be made, then, as the realistic-expectations section of the project plan predicted, the natural limit of this "blind polling + corrective hooks" approach may have been reached — in that case a real event-notification mechanism should be considered (e.g. the host ACTIVELY sending a vtest command when there is new data, instead of relying only on passive polling). This requires adding a NEW command to the vtest protocol (an asynchronous notification in the host→guest direction) — a bigger step, but a clear goal.

## Files used/updated
- `vendor/mesa/src/virtio/vulkan/vn_renderer.h`
- `vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`
- `vendor/mesa/src/virtio/vulkan/vn_ring.c`

## Note
Keep using `VIRGL_LOG_LEVEL=debug` — without it, the host-side `vkr_log` messages (including the CS error) are silently suppressed.
