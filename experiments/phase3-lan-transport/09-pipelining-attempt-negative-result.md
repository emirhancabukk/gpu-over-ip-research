# Phase 5 (early attempt) — Active Pull Throttling Gave No Measurable Benefit (2026-08-28, autonomous round 13)

> **Later note (2026-10-16):** the "real pipelining" this note calls a much bigger job was
> done in phase 4 with a one-call change — removing the unconditional push at submit
> time — and measured 435 s → 236 s over the real internet
> (`phase4-wan-aws/04-*.md`).

## The change tried

Following the suggestion in note 08: instead of `vn_ring_wait_seqno()` (`vn_ring.c`) doing
a real network round trip (a `sync_now` call) on every loop iteration, it was made to do one
only on EVERY 4th iteration (the `(pull_iter++ % 4) == 0` condition; the first iteration is
always included). Hypothesis: because `vn_relax`'s busy-wait phase (up to 256 fast
iterations) triggers a real network round trip on every iteration, under WAN latency this
phase itself may be unnecessarily expensive.

## Result: NO measurable benefit

Tested again with a 40ms artificial delay (the same `unshare --user --net` + `tc netem` method, see note 08):

| | Time |
|---|---|
| BEFORE the change (note 08) | 81.16s |
| AFTER the change (this round, 3 separate runs) | 80.95s, and two verification runs in the same range |

**No difference in practice.** Correctness was preserved (3/3 successful, `driverName = venus`), but there is no speed gain.

## Why it did not work (analysis)

Counting the `"client push"` lines in the log of one run found **289** separate push
operations (`grep -c "client push" ...`). Each one is a real network round trip on its own.
This shows my hypothesis was WRONG: the bottleneck is not `vn_relax` retrying UNNECESSARILY
again and again in its busy-wait phase — almost every `wait_seqno` call already succeeds on
the FIRST try (iter=0, the case where a real pull is always done regardless of the
throttle). So the extra iterations the throttle aimed to "save" practically never happened.

**The real cost item:** every Venus command itself already needs at least ~2-3 mandatory
round trips (active push = 1, the first/only necessary attempt of the active pull = 1, the
reply-pool synchronization if any = 1), and there are **many separate commands, such as 289**
(for `vulkaninfo --summary`'s full Vulkan instance + device query sequence). Total time ≈
(mandatory round trips per command) × (number of commands) × (RTT) — in an architecture that
is sequential/synchronous BY DESIGN this is an unavoidable cost, not waste that can be fixed
by "trying less".

## The right solution: real pipelining (a much bigger job)

For a real speed gain, IF POSSIBLE, several Venus commands need to be written to the ring
one after another without waiting for the host's reply to EACH of them separately, and sent
in bulk in ONE round trip (the host processes them in order and accumulates the results; the
client waits only once, for the completion of the LAST command). This requires redesigning
the calling pattern of the `vn_ring_submit_command`/`vn_call_*` layer (currently every
synchronous call does its own push + wait + reply-pull triple by itself), an engineering
change too big to be done safely within this round. It should be left as a separate, larger
phase 5 task; it cannot be solved with small adjustments such as a specific "throttle".

## Assessment and recommendation

Phase 3's main goal (proof of correctness) has already been completed solidly (notes 07 and
08). Performance optimization (real pipelining) is both a big engineering investment and
probably something that can be prioritized more deliberately once the real AWS WAN numbers
are in hand (phase 4) — "works but slow" has already been proven with a 10-40ms artificial
delay; making a big redesign investment before the real numbers (the real Turkey→Frankfurt
RTT) are known could be risky.

**Recommendation:** the autonomous loop can pause at this point / clearly summarize the
status to the user until the AWS quota status is clear (it requires a console check on the
user's side and could not be checked autonomously — the `aws` CLI is not installed on this
machine). If it continues, the next logical step could be either (a) starting the real
pipelining design (a big, multi-round job) or (b) waiting for the user to check and report
the AWS quota status themselves.
