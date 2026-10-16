# Phase 5 (early attempt #2) — "Push Only When Idle" Broke Correctness, Reverted (2026-08-28, autonomous round 14)

## Hypothesis and attempt

Continuing from note 09: the real bottleneck had been identified as each of the ~289
commands `vulkaninfo` produces having its own mandatory round trip. This round first
verified that the generated code of the Venus protocol really HAS the distinction between
`vn_async_vkFoo` (does not wait for a reply, "fire-and-forget") and
`vn_call_vkFoo`/`vn_submit_vkFoo` (waits for a reply) (e.g. `vn_async_vkCreateInstance`,
`vn_async_vkEnumerateInstanceVersion` — `vn_protocol_driver_instance.h`). Async calls are
submitted with `reply_size=0`, which means `vn_ring_submit_command()` NEVER calls
`vn_ring_wait_seqno()`.

**Problem identified:** the active push hook we added to `vn_ring_submit_internal()` (phase
3, round 11) runs unconditionally on EVERY submission (sync or async alike) — so even async
calls pay a synchronous round trip unnecessarily.

**Fix tried:** make the push conditional — force a push only when the host was idle
(`VK_RING_STATUS_IDLE_BIT_MESA`) when LAST observed; if the host is busy, assume the
background thread or a later call will deliver it anyway.

## Result: BROKEN, reverted

Test (0ms delay; just 3 runs were enough to show it): **3/3 runs timed out** (none finished
within 30s). Inspecting the log showed a clear vicious circle: it got stuck at `tail=228`; the
client logged the same tail value as a "push" again and again, but this data NEVER appeared
on the server side (the last reply data the host received always stayed old/the same).

**Root cause:** an async call (a) skips the push (on the assumption that the host was not
idle) AND (b) never calls `vn_ring_wait_seqno` (because it is async) → this command's data
stays written in LOCAL memory, but NO mechanism delivers it to the host. Normally the
background poll thread would have caught it sooner or later, BUT in the Bug #8 fix in note 07
the background thread had been COMPLETELY disabled for the ring (so that only the active
push/pull hooks would be relied on). These two changes (Bug #8's "the background thread must
not touch the ring" decision + this round's "do not push unless idle" attempt) TOGETHER create
a gap in the delivery guarantee: no path guarantees that the data is sent to the host.

**Fix:** the push was made unconditional again (`vn_ring.c`, `vn_ring_submit_internal`). 3/3 verification runs are clean again (`driverName = venus`).

## Takeaway

The ring's synchronization architecture currently depends **entirely** on the active
push/pull hooks (the background thread is disabled) — this means EVERY submission has to be
responsible for its own delivery; the assumption "somebody will see it anyway" is NOT SAFE.
A real optimization needs one of two options:

1. **Re-enable the background thread for the ring** (as a fallback/safety net, but this time without re-triggering the cause of Bug #8 from note 09 — e.g. with a much LONGER period, e.g. 5-10ms, only as a backup "in case someone forgets", infrequent enough not to race with the active hooks).
2. **Proper pipelining:** REALLY accumulate async calls (without pushing) and, when the next SYNC call comes (which will wait anyway, so its push is mandatory), send them all in ONE round trip. This means not "never push" but "defer and push in BULK" — more complex than option 1, but with a larger potential gain (N async calls = still only the 1 push of the last sync call).

Outside this round's scope (for reasons of time/complexity) — this is a concrete, clear starting point for the next round.

## Concrete Task for the Next Round

1. **Try option 1 (low risk, quick test):** re-enable the background thread for the ring with a long period such as 5ms (ONLY a fallback; the active hooks remain the primary mechanism). Measure correctness and the tc netem 40ms time.
2. If time allows, start thinking about option 2 (real deferral + bulk push) — but because `vn_ring_submit_internal` would need to know "async or sync", this requires passing a parameter down from the upper layers (`vn_ring_submit_command`), a more invasive change.
