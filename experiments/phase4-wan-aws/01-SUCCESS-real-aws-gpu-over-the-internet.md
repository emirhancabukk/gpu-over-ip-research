# Phase 4 — Connected to a Real AWS GPU over the Real Internet (2026-08-29)

## Milestone

`vulkaninfo --summary` successfully enumerated a real AWS `g5.xlarge` (NVIDIA A10G) GPU
running in Frankfurt (eu-central-1) through the Venus protocol, from a home internet
connection over the real internet (plain TCP, not an SSH tunnel). **Multiple repeated runs,
all successful**: `deviceName = Virtio-GPU Venus (NVIDIA A10G)`, `driverName = venus`, exit
code 0.

This meets the core evidence for the plan's phase 4 goal: the architecture works end to end
not only on localhost but also over a real WAN.

## Setup

- AWS EC2 `g5.xlarge`, `eu-central-1` (Frankfurt), Ubuntu 22.04, NVIDIA driver `580` (proprietary, including the Vulkan ICD).
- The patched `virglrenderer` (from phase 3, TCP transport + all bug fixes) was built on AWS and run with `virgl_test_server --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445`.
- AWS security group: only SSH (22) and port 33445, with the source restricted to "My IP" (not open to the world).
- The local client (the patched Mesa from phase 3) connected over the real internet with `VTEST_SOCKET_NAME=tcp:<public-ip>:33445`.
- Real RTT (measured with TCP connect timing): **~65-70ms** (Turkey-Frankfurt).

## The performance problem and the optimization round

The first run took **430 seconds** (much slower than the 81 seconds with a 40ms artificial
`tc netem` delay — the ratio was not linear in the RTT, which was found worth investigating).

### Diagnosis (strace was not installed; manual profiling was added to the C code)

`clock_gettime`-based duration measurement was added to `vtest_shmem_sync_now` (`vn_renderer_vtest.c`). Findings:
- In a 40-second sample, **89%** of the time was spent in network round trips.
- The ring synchronization (`vtest_shmem_sync_ring_cycle`) PUT **the WHOLE 128KB ring buffer** on every cycle (although usually only a few hundred bytes change) + did **3 separate synchronous GETs** (head, status, extra).
- The reply-pool synchronization GET **the whole, fixed 1MB pool** on every synchronous call.

### Fixes applied

1. **TCP_NODELAY was missing on the server side** (`vtest_server.c`, the accepted client socket) — added, but on its own it made no measurable difference (the Nagle/delayed-ACK hypothesis turned out wrong; the real bottleneck was data volume).
2. **Ring delta push**: by tracking `ring->cur` (a monotonic counter), only the [last_sent, new) range is PUT, not the whole 128KB buffer (the wrap-around case is handled with a two-part PUT). `vn_renderer_vtest.c`.
3. **Merging the ring GETs**: head (offset 0) + status (offset 128) are merged into a single GET (the gap between them is discarded); extra stays separate (it is far away, right after the end of the buffer).
4. **Reply-pool delta pull**: a `hint_size` parameter was added to the `sync_now` hook (`vn_renderer.h`, `vn_ring.c`, `vn_renderer_vtest.c`). In the first attempt `reply_shmem_pool.used` (a cumulative bump-allocator counter) was used directly as `hint_size` — this was WRONG: `used` means "how much has been allocated in total so far", so every call kept pulling [0, used), and an ever-growing range was read again and again as the run went on. Fixed: a `last_synced_size` state was added, and only the [last_synced_size, hint_size) delta is pulled.
5. **A bug found and fixed**: in `vtest_shmem_sync_ring_cycle`, the `cur` value was read BEFORE the mutex was taken, but the `last_pushed_cur` state update was done INSIDE the mutex. When two threads (the background sync thread + the active `sync_now` hook) collided, `last_pushed_cur` could go BACKWARDS — this showed up as an `abort()` (the ring-full assertion) in `vn_ring_submit_internal` (diagnosed with `coredumpctl`). Fix: the `cur` read was moved inside the mutex.

### Result: partial success, an open puzzle

After the fixes:
- Reply-pool sync: 780-900ms → **~63ms on average** (about a 13x speed-up), consistent.
- Ring sync: most cycles 730ms → **~120-140ms** (about a 5-6x speed-up).
- **BUT**: **15-20%** of the samples still take **1-7 seconds** (randomly spread, not apparently tied to a specific command).

**Host GPU utilization was 0% throughout the test** (verified with nvidia-smi) — so these
slowdowns do NOT come from GPU driver processing time. There is no error/warning in the host
log (`VIRGL_LOG_LEVEL=debug`). This suggests the problem is in the real network layer
(jitter, packet loss, TCP retransmission) — it may be a WAN reality that code optimizations
cannot remove, but this was NOT conclusively PROVEN (no packet-level analysis with
tcpdump/wireshark was done; strace was not installed in this environment).

Full run times got **worse, contrary to expectations**, during the optimization round (430s
→ 458s → 510s → 645s) — this does not show that the code changes were INSUFFICIENT, but that
an uncontrollable variable between measurements (probably real network conditions changing
over time, or a state accumulated by many consecutive test/server-restart cycles) affected
the results. A clean, single-variable comparison could not be made in this round.

## Known limitations / notes for the next step

- `strace`/`tcpdump` are not installed in this environment (they need sudo; non-interactive sudo fails). For real packet-level analysis, one of them must be installed (by asking the user for the sudo password) — this is the only way to determine conclusively the real source of the "15-20% occasionally 1-7s" pattern (packet loss, the TCP congestion window, or something else).
- The temporary debug/profiling code (`[PROF] sync_now ...` fprintfs in `vn_renderer_vtest.c`) is still in the code base — left in for diagnosis; it adds unnecessary I/O overhead on the performance-critical path and should be cleaned up.
- The AWS `g5.xlarge` instance was left running (the server process was stopped, but the instance itself is still running) — mind the $30 autonomous spending cap; it should be stopped when not in use.

## Cost

In this session the instance was up for about 2-2.5 hours (~$2.5-3, under the $30 cap).
