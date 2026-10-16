# Phase 5 — Root Cause: The Background Thread Wastes 258 MB of Transfer per Run (2026-09-04)

## Summary

Note 06's conclusion, "the slowness is caused by the throughput capacity of the local
internet connection", is **most likely wrong**. The measurement shows that the client is
filling its own download capacity: in a 266-second run, the reply-pool background thread
pulled **the full 1 MB 258 times** (258 MB), while the useful data in the same run was
**1.5 KB**.

## Setup

The same AWS `g5.xlarge` (Frankfurt, NVIDIA A10G, driver 580.173.02), with a new public IP.
The server binary is from 29 August and was not rebuilt. The three source files on the server
(`vtest_server.c`, `vtest_renderer.c`, `vkr_ring.c`) were verified to be **byte-for-byte
identical** to the version-controlled copies in the repository — the experiment is
reproducible.

The security group's 33445 rule was pinned to the old home IP; after it was updated to the
current home IP (`/32`), the connection opened.

TCP-connect RTT before and after the run: median **89-91 ms**, spread **27-30 ms** (a
median of 66.1 ms / a spread of 3.3 ms had been measured to the regional endpoint on 31 August).

## Run result: crashed

`vulkaninfo --summary`, `exit=134` (SIGABRT), **266 seconds**, without producing any output.
282 `sync_now` calls had completed (251 ring + 31 reply pool). It did not reach the previous
round's 353 ring syncs.

Coredump backtrace (`coredumpctl info 53468`):

```
abort
vtest_read                                  <-- here
vtest_vcmd_transfer_get2
vtest_shmem_sync_ring_cycle
vtest_shmem_sync_now
vn_renderer_shmem_sync_now
vn_ring_submit_command
vn_submit_vkGetPhysicalDeviceImageFormatProperties2
```

It is **not** the ring-full assertion; the socket read failed. The last line of the server
log is `client: VTEST_CLIENT_ERROR_INPUT_READ` — so first the server's read from the client
failed, the server closed the connection, and then the client's read blew up.

Note: the name `vtest_vcmd_transfer_get2` in the backtrace does not mean a return to dead end
#15 (the GET2/PUT2 opcodes) — it is the name of the client-side helper function; the opcode it
sends is still the old 4.

## Measurement: the breakdown of the traffic

The server log (`/home/ubuntu/vtest-20260904-123435.log`) was counted:

| Source of the reply-pool `transfer_get` | Count | Transfer size |
|---|---|---|
| Background thread | **258** | `offset=0 size=1048576` (the full 1 MB) |
| Active `sync_now` hook | 31 | 24-64 bytes, increasing offset (the delta works correctly) |

The `size` field printed by the server's `fprintf` is `data_size`, the actual requested
transfer size — not the resource's total size (`vtest_renderer.c:1656`). So `1048576` is the
number of bytes actually transferred.

The number 31 is identical to the number of `is_ring=0` lines counted from the `[PROF]` lines
on the client. So **only the active hook** uses the delta path; the background thread passes
no `hint_size`, so it never enters the delta branch in `vtest_shmem_sync_now`
(`hint_size > 0 && hint_size <= mmap_size`) and falls into the full-pool GET in the final
`else` branch.

**258 MB / 266 s ≈ 7.8 Mbps of continuous, entirely wasted download.**

## Why this invalidates note 06's diagnosis

Note 06 reached the conclusion "the problem is neither in the code nor on the host; the only
remaining explanation is the throughput capacity of the local connection" from three
observations: the RTT constant at ~72-75 ms, the host completely idle, the code returned to its
known-good state.

All three observations are correct, but the conclusion is incomplete. When a continuous 7.8
Mbps of useless download saturates a home connection, the picture that results is exactly this:

- **Small TCP-connect probes are not affected** — tiny packets get through the queue, so the RTT measurement looks constant.
- **Real data transfers collapse** — ring syncs went from 174-201 ms (2 RTT, the design's theoretical floor) measured at the start of the run to **755-1051 ms** by the time of the crash.
- **The host looks idle** — the GPU really is not working, because the bottleneck is the network.

So the "not the code" inference came from the code that was looked at not being the code that
was measured: the culprit is not `sync_now`'s profiled active path but the background thread,
which was never profiled.

Note 01's finding "15-20% of the samples take 1-7 seconds, randomly spread, GPU 0%" is a
symptom of the same mechanism: a 1 MB download starting every 100 ms makes a ring sync that
lands in that interval wait in the queue.

This also explains why optimization #4 (the thread period 2 ms → 100 ms, "a ~2.8x speed-up")
worked: it did not fix the root cause; it made the full-pool pulls 50 times less frequent. At a
2 ms period the demand would be ~500 MB per second, which is impossible — that is also the
cause of the 780-900 ms reply-pool syncs in the first measurements.

## This was already known, but not measured

The comment around `vn_renderer_vtest.c:729` describes the situation correctly: *"the
reply-pool branch above does a BLIND, full 1MB GET every time it runs — unlike the ring, it has
no delta tracking ... over a real WAN link this was a large, continuous, mostly-wasted stream of
traffic"*.

What was missing was the **measured size** of this, and seeing that it **was the cause of
the slowness in note 06**. The comment recorded it as an inefficiency, not as the root cause of
the stall.

## The direction of the fix, and why it is not "just delete it"

The same comment also records a dead end: the attempt to **remove this branch entirely was
reverted** — only 8 synchronous calls in 10 minutes, ~485 ms on average (it was ~63 ms with the
100 ms version), while the RTT had been re-measured at the time and verified unchanged. The
root cause was not isolated. So the active hook alone is not a sufficient replacement, and the
reason is still unknown.

So the fix should not be removing the thread but **limiting how much it pulls**: staying at
the same 100 ms period and pulling a bounded window (e.g. 4-8 KB) starting at
`shmem->last_synced_size` instead of `offset=0 size=1048576`. A reply is small; this branch's
job is only to catch the rare case the active hook missed. Expected effect: ~4 KB per 100 ms
instead of 1 MB per 100 ms, a ~250x reduction, with the safety net preserved.

Because the mechanism that broke the removal attempt is still unknown, not changing the period
or the existence of the branch is a deliberate choice — let the transfer size be the only
variable that changes.

## Falsification test

This hypothesis can be refuted cheaply: repeat the same run after the bounded-window change
above. If the hypothesis is right, the `size=1048576` lines disappear from the server log, ring
sync times stay in the 174-201 ms band throughout the run (instead of climbing to 750 ms+
towards the end), and the run completes without crashing. If the hypothesis is wrong, the
slowdown continues regardless of the transfer size.

## Also found: `vtest_read` is fragile on its error path

```c
const ssize_t ret = read(vtest->sock_fd, ptr, size);
if (unlikely(ret < 0)) { vn_log(...); abort(); }
ptr += ret; size -= ret;
} while (size);
```

Two problems:

1. `ret < 0` → straight to `abort()`. A transient TCP error kills the whole run; there is no
   retry and no clean error return. For a system running over a WAN, this means long runs are
   structurally fragile.
2. `ret == 0` (the other side closed the connection cleanly) is not treated as an error: since
   `size` never decreases, the loop runs forever — a busy wait on EOF.

This is a separate robustness job, independent of the root cause.

## Status (updated 2026-10-16)

- The bounded-window fix was applied and built cleanly on 4 September, after this note was
  written; on 16 October it was committed as `patches/mesa/0002-*.patch`. **The verification
  run has not been done yet** — the falsification test above is still open.
- Lesson: if a security group rule is pinned to a home IP, the connection breaks silently when
  the IP changes — you see a `timeout`, not `connection refused`.

## Later note (2026-10-16): why netem could not see this problem

The netem tests in phase 3 (`phase3-lan-transport/08-*.md`) added only delay (`netem delay
<N>ms`); there was no bandwidth limit. Without a limit, a 1 MB pull finishes in a few round
trips; on an 8 Mbps home connection it takes ~1 second and holds the single socket (and
`sock_mutex`) for that whole time. That is why the 258 MB problem never showed up in local
tests: netem models latency correctly and bandwidth not at all.

The good part: the falsification test does not need AWS. Adding a bandwidth limit to netem
(for example `netem delay 35ms rate 10mbit`) imitates the home connection locally; the client
with and without the fix can be compared under the same conditions.

## Later note (2026-10-16): the whole run, from the client's own log

The client's stderr from this run is in
[`07-data-client-stderr-2026-09-04.txt`](07-data-client-stderr-2026-09-04.txt): 282 `[PROF]`
lines, one per `sync_now` call (251 ring, 31 reply pool), plus 4 `[DEBUG]` lines. Read over the
whole run, it refines the "174-201 ms at the start, 755-1051 ms at the crash" summary above:

| Fifth of the run | Ring syncs | Median | 10th percentile | 90th percentile | Slower than 600 ms |
|---|---|---|---|---|---|
| 1 | 50 | 201 ms | 170 ms | 801 ms | 42% |
| 2 | 50 | 748 ms | 173 ms | 1,015 ms | 78% |
| 3 | 50 | 817 ms | 695 ms | 1,050 ms | 94% |
| 4 | 50 | 794 ms | 722 ms | 1,017 ms | 100% |
| 5 | 51 | 865 ms | 750 ms | 1,051 ms | 100% |

The ring syncs fall into two clusters: 39 (16%) under 250 ms, with a median of 175 ms (about
two round trips), and 208 (83%) over 600 ms, with a median of 802 ms. Slow syncs were already
there from the start: the first ten were 443, 170, 984, 186, 842, 184, 757, 181, 797 and
180 ms. This alternation was not investigated; one explanation consistent with it is ring syncs
queueing for the single socket behind a background 1 MB pull. The fast cluster disappears as the
run goes on.

Reply-pool syncs over the run: median 546 ms, minimum 80 ms, maximum 1,247 ms.

`[PROF]` times only `sync_now` calls; the background thread's own transfers are not timed, so
the duration of a single 1 MB pull is not measured directly.
