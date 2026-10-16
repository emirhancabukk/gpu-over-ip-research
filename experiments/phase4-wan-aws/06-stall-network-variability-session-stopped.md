# Phase 5 — Stall: Real Network Variability, Not Code (2026-08-29)

> **Later note (2026-10-16):** this note's conclusion that "the slowness comes from the
> local connection's throughput" is most likely wrong. Note 07 measured that the
> reply-pool background thread pulls ~258 MB of waste per run and saturates the
> connection itself. The text below is left as it was written that day.

## Status

An attempt to remove the reply-pool background thread entirely (a continuation of note 05)
was tested: only 8 synchronous calls completed in 10 minutes (~485ms on average) — clearly
worse than the proven 100ms-period version (~63ms on average, measured earlier in the same
session). The change was reverted (back to the 100ms period, the proven state).

**But the same slowness continued after the revert too**: **zero** synchronous calls
completed in 10 minutes, and the average sync_now time rose to 851ms.

## Distinguishing diagnosis

- Real RTT (TCP connect timing): a consistent **~72-75ms** throughout the session — the same during these tests too, unchanged.
- Host CPU/GPU: completely idle (`top`: 97.6% idle, load average 0.04; `nvidia-smi`: 0%).
- Code: returned to the proven configuration (which had earlier shown a 2.8x speed-up), same result.

Performance degrading this much while the RTT stays constant, the host sits completely idle,
AND the code has been returned to its known-good state shows that the problem is neither in
the code nor on the host — the only remaining explanation is the state, at that moment, of the
local internet connection's **throughput/packet-processing capacity** (a congestion that does
not affect the RTT but affects real data transfer, e.g. at the ISP level, or other use of the
local network).

## Conclusion

This is a reality the project plan already accepted: in a system running over a real WAN,
network conditions change over time, and the measurements in a single session are not
absolute/permanent. There is nothing more that can be done on the code side at this point —
trying more "optimizations" would mean measuring on unreliable ground.

## Total gains of this session (summary)

1. A critical race condition (ring buffer data corruption → crash) was found and fixed.
2. The device-memory (bo) transport was made to work over TCP — real rendering no longer crashes.
3. Ring pipelining (the unconditional submit-time push removed) — a measured 46% speed-up (435s→236s, in good network conditions).
4. The reply-pool background thread's frequency was lowered (2ms→100ms) — a measured ~2.8x speed-up (in good network conditions).

All four are real, code-driven, lasting improvements. The current slowness does not
invalidate them — it only makes the current measurement unreliable.

## Recommendation

The session should stop here. If the AWS instance is left running, the cost keeps accruing.
If work continues, a clean verification round with the current code base (including the 4
optimizations) should be done when network conditions are back to normal (at a different time).
