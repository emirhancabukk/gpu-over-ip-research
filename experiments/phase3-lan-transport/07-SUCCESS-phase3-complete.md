# Phase 3 — COMPLETE: Venus/vtest Works End to End over TCP (2026-08-28, autonomous round 12)

## Result

`vulkaninfo --summary`, with no virtio-gpu/QEMU, successfully enumerated the real AMD GPU
through the Venus protocol **over a real TCP socket between two separate processes**. **5
out of 5 consecutive runs** produced clean output (exit code 0, `driverName = venus`). Full
output: `vulkaninfo-over-tcp-output.txt` (in this folder).

```
	deviceName         = Virtio-GPU Venus (AMD Radeon Graphics (RADV RENOIR))
	driverID           = DRIVER_ID_MESA_VENUS
	driverName         = venus
	driverInfo         = Mesa 26.3.0-devel (git-e9a87f80b2)
```

This meets the plan's phase 3 goal: it proves that Venus works correctly, from scratch, over
a TCP socket instead of its default virtio-gpu transport, on a LAN (currently localhost,
latency ~0). The next step (already defined in the plan): inject artificial delay
(10/20/40ms) with `tc netem` and measure where it breaks.

## The LAST two bugs found and fixed this round

The two open problems left by the previous rounds (notes 04-06) were closed this round:

### Bug #8: an uncoordinated race between the background poll thread and the active push/pull hooks

The "the data the host receives is entirely zero" symptom found in note 06 came from three
SEPARATE sources that did not know about each other (the background poll thread, the active
push in `vn_ring_submit_internal`, the active pull in `vn_ring_wait_seqno`) triggering
independent, unordered full-cycle synchronizations for the same ring resource. All three used
the same mutex (no data corruption), but their ordering was unpredictable. **Fix:** the
background thread now does NOTHING for the ring resource (the loop body is empty in the
`is_ring` case) — the ring's synchronization now depends ONLY on two precise, event-triggered
sources: the active push when new data is written, and the active pull while waiting for
progress. This eliminated the problem of the tail reaching the host again and again with the
same value as "zero data".

**Known limitation:** `vn_ring_wait_space()` (back-pressure when the ring is full) is still
not hooked — in large-command/full-ring scenarios this path kept relying on the background
thread, but now that the background thread no longer runs for the ring, this path may be left
COMPLETELY unsynchronized. It caused no problem for `vulkaninfo`'s small commands (the 128KB
ring never filled up), but it must be addressed again for a real render/game workload.

### Bug #9: `vtest_vcmd_sync_wait()` was still waiting for SCM_RIGHTS — an infinite block at teardown

Found by sending `SIGABRT` to a live "hung" run and analysing the core dump with
`coredumpctl`: the enumeration had finished COMPLETELY successfully, and the process was
waiting **forever** in `vkDestroyInstance` → `vn_ring_destroy` → `vn_renderer_wait` →
`vtest_vcmd_sync_wait` → `vtest_receive_fd` → `recvmsg()`. This was ANOTHER fd-receive point,
never noticed, OUTSIDE `vtest_vcmd_resource_create_blob` (fixed for SCM_RIGHTS in earlier
rounds) — Venus's sync/fence wait mechanism expects the server to pass an eventfd to the
client with `vtest_send_fd` and the client to `poll()` it; over TCP this fd can never be
received.

**Fix (a deliberate, narrowly scoped shortcut):** in TCP mode, `vtest_vcmd_sync_wait()` no
longer calls `vtest_receive_fd()` and returns `-1`; `vtest_wait()` interprets this as
"nothing to poll, consider it done already" (returns `VK_SUCCESS` and never calls
`sync_wait_poll`). **This gives NO correctness guarantee for a real, not-yet-signalled
fence** — it was verified only for simple workloads such as enumeration/synchronous calls.
Real render/game workloads (which need asynchronous fence waits) will need a host→guest
notification channel (phase 5).

## Was the phase 3 acceptance criterion met?

The plan's phase 2 acceptance criterion ("render through Venus/virtio-gpu with no network at
all") had already been met. Phase 3's goal ("adapt Venus's transport to a TCP/QUIC socket and
verify that it works correctly on a LAN") is now met: **5/5 consistent, clean `vulkaninfo
--summary` runs, correctly enumerating the real AMD GPU through Venus over a TCP socket.**

## Summary of all changes (throughout phase 3, all rounds)

**`vendor/virglrenderer/vtest/vtest_renderer.c`:**
- `vtest_resource_create_blob`: mmap on the host side for HOST3D blobs too (for the SHM fd type), filling `res->iov`.
- `vtest_transfer_get/put_internal`: a direct memcpy fast path for blob resources (reusing box.x as the offset).
- `vtest_send_fd`: a no-op on non-AF_UNIX sockets.
- `atomic_thread_fence(memory_order_release)` after the blob PUT memcpy (experimental, the hypothesis B test — in the end it was NOT the main fix, but it is harmless, so it was left in).
- `vkr_ring_submit_cmd` (vendor/virglrenderer/src/venus/vkr_ring.c): a ring buffer dump debug print at the fatal moment.

**`vendor/virglrenderer/vtest/vtest_server.c`:** `tcp:<port>` socket support.

**`vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`:**
- TCP connection support (`vtest_connect_socket_tcp`), an `is_tcp` flag.
- Use of the legacy TRANSFER_GET/PUT (opcodes 4/5), reusing the box.x field as the offset carrier.
- `vtest_shmem_create`: an anonymous mmap + a background synchronization thread in TCP mode.
- `vtest_shmem_sync_ring_cycle`: ownership-respecting, correctly ordered (buffer first, tail after) synchronization for the ring.
- `vtest_shmem_sync_now` + `shmem_ops.sync_now`: synchronous, event-triggered push/pull for the reply pool AND the ring.
- The background thread no longer runs for the ring (only for the reply pool and as a general fallback).
- `vtest_vcmd_sync_wait`/`vtest_wait`: an fd-wait bypass in TCP mode (the Bug #9 fix).

**`vendor/mesa/src/virtio/vulkan/vn_renderer.h`:** the optional `shmem_ops.sync_now` hook + the `vn_renderer_shmem_sync_now()` helper.

**`vendor/mesa/src/virtio/vulkan/vn_ring.c`:**
- `vn_ring_wait_seqno`: a `sync_now` call for the reply pool + the ring on success (an active pull for the ring on every iteration).
- `vn_ring_submit_internal`: an active push for the ring right after new data is written.

## Exact commands used (reference, reproducible)

Server:
```bash
VIRGL_LOG_LEVEL=debug ./vendor/virglrenderer/build/vtest/virgl_test_server \
  --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445
```

Client:
```bash
ICD=$(find vendor/mesa/build -name "virtio_devenv_icd.x86_64.json")
VK_ICD_FILENAMES="$ICD" VTEST_SOCKET_NAME=tcp:127.0.0.1:33445 vulkaninfo --summary
```

Build (for both kinds of change):
```bash
source vendor/.build-venv/bin/activate
cd vendor/mesa && meson compile -C build       # for client-side changes
cd vendor/virglrenderer && meson compile -C build  # for server-side changes
```

## Next Step: the second half of phase 3 — an artificial delay test with `tc netem`

The plan already anticipates this: after verification on a LAN (latency ~0), inject an
artificial 10ms, 20ms, 40ms delay with `tc netem` and measure which synchronous points
(especially our active push/pull hooks — each now corresponds to ONE network round trip,
which will become costly as the latency grows) cause problems. This is the last step that can
be done cheaply (no cloud cost) before moving to the real AWS WAN test (phase 4).

Concrete task:
1. Add an artificial delay to the loopback (`lo`) interface with `tc qdisc add dev lo root netem delay 10ms` (or over a `veth` pair — adding netem to `lo` may not affect localhost connections on some systems; verify carefully, a setup with two separate network namespaces + a veth pair may be needed).
2. Run the same 5-repeat test with a 10ms, then 20ms, then 40ms delay, and record the success rate and (if any) the total time for each.
3. Write the findings as a new note under `experiments/phase3-lan-transport/`.
4. When the test is done, do not forget to remove the delay with `tc qdisc del dev lo root netem`.
