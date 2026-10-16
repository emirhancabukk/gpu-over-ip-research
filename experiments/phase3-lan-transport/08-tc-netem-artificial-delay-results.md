# Phase 3 — tc netem Artificial Delay Test: 10/20/40ms All SUCCESSFUL (2026-08-28, autonomous round 12 continued)

> **Later note (2026-10-16):** these tests added delay only, with no bandwidth limit. They
> are a valid predictor of the effect of latency, but they cannot see bandwidth-bound
> problems — the 258 MB of wasted transfer found in phase 4 (`phase4-wan-aws/07-*.md`)
> never showed up here for exactly this reason. New tests should add
> `netem ... rate <N>mbit`.

## Method: an isolated network namespace, no root needed

`tc qdisc add dev lo ...` normally needs `sudo`/root (tried: `sudo -n` asked for a password,
so the autonomous loop could not continue). Instead, **an unprivileged user + network
namespace** was used:

```bash
unshare --user --net --map-root-user sleep 3600 &   # the "anchor" process that keeps the isolated ns alive
nsenter --target <pid> --net --user --preserve-credentials bash -c "
  ip link set lo up
  tc qdisc add dev lo root netem delay <N>ms
  # ... run the server and the client inside the SAME nsenter too ...
"
```

This allowed testing WITHOUT touching the real host's network stack at all (inside a
completely isolated namespace with its own `lo`) — fully in line with the plan's constraint
"nothing with a lasting system-wide effect will be done on the bare host", and even safer
than `sudo` (ZERO effect on the real system; the namespace is cleaned up automatically when
its last process exits).

## Results

| Artificial delay (one way) | Result | Time (full `vulkaninfo --summary` run) |
|---|---|---|
| 0ms (the basic phase 3 test, previous note) | ✅ SUCCESS (5/5) | ~a few seconds |
| 10ms | ✅ SUCCESS | 20.75s |
| 20ms | ✅ SUCCESS | 40.82s |
| 40ms | ✅ SUCCESS | 81.16s |

**All produced clean output with `driverName = venus`; no crash/assertion/CS error at all.**

## Interpretation

1. **The correctness side looks solid.** From 0 to 40ms (a realistic RTT range for a real AWS region such as Frankfurt from Turkey) there was no corruption/crash. This shows that the main findings of phase 3 this round (ring synchronization, the reply-pool freshness hook, the sync-wait bypass) really are solid, not just "a fragile demo that works in the best case".

2. **Time scales LINEARLY with the delay** (10ms→20.75s, 20ms→40.82s, 40ms→81.16s — roughly 2x delay = 2x time). This shows that the system makes a constant number of SEQUENTIAL round trips (not parallel/pipelined) — an expected result, because this round's active push/pull hooks (`vn_ring_submit_internal`, `vn_ring_wait_seqno`) need separate, sequential network round trips for EVERY synchronous Venus call, and the background poll thread no longer runs for the ring at all (see note 07, the Bug #8 fix).

3. **A realistic figure:** for a 40ms one-way delay (about an 80ms RTT, a bad WAN scenario), `vulkaninfo`'s ENTIRE enumeration took ~81 seconds. For a real game/render workload this would be UNACCEPTABLY slow — but this is an OPTIMIZATION problem the plan anticipated from the start and the subject of phase 5, not an architectural blocker. The current design is "stop at every step, do a round trip, continue" — a real production design should PIPELINE the commands (send several commands one after another without waiting for the host to reply, and process the replies in bulk), exactly as phase 1's early findings predicted.

## Next Steps

- **Phase 3 is now COMPLETELY finished** (LAN + artificial WAN delay, all verified).
- **The decision to move to phase 4 should be left to the user**: it is still unknown whether the AWS quota for the real AWS GPU test (g5.xlarge, A10G) has been approved (it was rejected a second time in earlier rounds) — this is not something the autonomous loop can check; it needs an AWS console check on the user's side.
- **The next technical step, which can be thought of as an early part of phase 5** (before performance optimization): pipelining — instead of `vn_ring_wait_seqno` doing a full round trip on EVERY iteration, check several pending commands in bulk in a single round trip. This could greatly reduce the effect of WAN latency (~1 round trip instead of N round trips for N commands).
