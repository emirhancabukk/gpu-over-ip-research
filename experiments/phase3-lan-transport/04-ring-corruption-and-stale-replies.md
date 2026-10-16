# Phase 3 — Root Cause of the Ring Corruption Found and Fixed; Remaining Problem Is a Reply-Pool Freshness Race (2026-08-27/28, autonomous round 9)

## Summary of this round

Continued from where the previous round stopped (`vn_relax`'s timing budget looked
insufficient). The real root cause turned out not to be timing but **the ring
synchronization corrupting itself** — three separate, real bugs were found, and the first
two were fixed completely:

1. **Blind bidirectional full-buffer copying was overwriting the ring's head/tail/status fields** → fixed (field-by-field, ownership-aware synchronization)
2. **The PUT order was wrong (tail was sent before the buffer)** → fixed (buffer first, tail after)
3. **No freshness/happens-before guarantee for the reply pool** → **partly fixed, still fails from time to time** (the open problem left by this round)

## Bug #5: the ring's head/tail/status fields were overwritten by blind copying

`vn_ring_get_layout()` (`vn_ring.c:257`) defines the ring's shmem layout like this (for a
128KB buffer + 4B extra, with the fixed `buf_size=128*1024, extra_size=4` in `vn_instance.c`):

```
[0]      head   (4B)  — HOST-owned (the host advances it while consuming)
[64]     tail   (4B)  — GUEST-owned (the guest advances it while producing)
[128]    status (4B)  — HOST-owned (e.g. a periodic ALIVE signal)
[192]    buffer (128KiB) — GUEST-owned (the actual command data)
[131264] extra  (4B)  — HOST-owned
total shmem_size = 131268 (matches the ring blob size we observed exactly)
```

The previous round's synchronization mechanism copied **all 131268 bytes** on every cycle,
first as a PUT (guest→host) and then as a GET (host→guest). This made every guest PUT
**overwrite the current values in the host's HEAD/STATUS fields with the guest's stale
local copy** — the host's progress (head advancing, the ALIVE bit being set) was silently
undone on the guest's next PUT. As a result `vn_relax` could never see the host's real
progress and aborted when its budget ran out — this is what the previous round took for a
"timing problem".

**Fix:** added `vtest_shmem_sync_ring_cycle()` (`vn_renderer_vtest.c`) — it now PUTs only
the guest-owned fields (tail + buffer) and GETs only the host-owned fields (head + status +
extra). Because the legacy PUT/GET command has no real offset field
(`vtest_transfer_decode_args` always set `offset=0`), we reused the unused `box.x` field as
the offset carrier — on both the client (`vn_renderer_vtest.c`) and the server
(`vtest_renderer.c`) side, affecting only our blob fast path (without touching the classic
texture transfer_3d path).

## Bug #6: the PUT order — tail was sent BEFORE the buffer

Right after the field-by-field fix, a **new** error appeared on the host side: `vkr:
vkCreateInstance resulted in CS error` / `vkr: ring_submit_cmd: vn_dispatch_command failed`
(the host saw a corrupted/incomplete command). Root cause: our synchronization loop PUT the
TAIL (4B) first and then the BUFFER (128KB). On the host, the ring consumer thread watches
for tail changes independently and starts decoding the current buffer contents
**immediately** when the tail changes — if the tail is published before the buffer data,
the host can read buffer bytes that are not yet updated/stale (the classic "lock-free SPSC
ring" data race pattern).

**Fix:** the order was reversed — the BUFFER is PUT first, then the TAIL. After this fix
the host side runs **completely clean**: `vkCreateInstance` is processed successfully, and
the `CS error` was never seen again.

## Bug #7 (partly fixed): the reply-pool freshness/happens-before race

After these fixes, a NEW error appeared on the client side:
`vn_decode_vkEnumerateInstanceVersion_reply: Assertion command_type ==
VK_COMMAND_TYPE_vkEnumerateInstanceVersion_EXT failed` — the expected value is 137, but in
some runs the value read was 0 (all zero bytes).

The chain of root causes:

1. `instance->reply_shmem_pool` (`vn_instance.c`) is a fixed **1u<<20 (1048576) bytes** — the pool carrying the replies to synchronous calls. The host is the **only** side that writes replies into this memory; the guest only reads (the pool bookkeeping is entirely client-side and not stored in the blob). The earlier blind bidirectional copying triggered the same self-inflicted bug here too: the guest's PUT kept overwriting the real reply the host had just written. **Fix:** added detection specific to this resource size (`size == 1u<<20`); it is now **GET only**, never PUT.

2. Fixing that revealed a NEW error: in the server log, the ring synchronization thread and the reply-pool synchronization thread share the SAME `vtest->sock_mutex`, and the pattern of a bare `thrd_yield()` followed by locking again immediately created **a classic "lock convoy"** on Linux's unfair (non-FIFO) pthread mutex — during a 20-second run the reply-pool thread **never won the mutex even once**, so even though the host reply was written correctly on the server side, the client never saw it (it always read the initial zeros).
   - **Partial fix:** a short real `nanosleep` (200µs) was added at the end of the ring loop, and the reply-pool thread was given a longer sleep (2ms) — to keep the ring prioritized while giving the reply pool a chance to "catch up later".
   - With this fix, **a fully correct result was observed at least once**: `got command_type=137` (matching the expected value EXACTLY) — so the mechanism **can work correctly**, proven.
   - However, it is **still not deterministic** — in consecutive runs it sometimes still reads `command_type=0` (there is still no guaranteed "happens-before" relationship between the host writing the reply and our last GET, only a statistically improved timing).

## Overall Assessment

This round produced the project's most important architectural finding: **blind,
bidirectional, full-buffer copying is a fundamentally wrong synchronization strategy for a
producer/consumer ring buffer** — it corrupts itself. Ownership-respecting, order-aware
synchronization (applied this round) solves it, and **a real Venus command (vkCreateInstance
+ vkEnumerateInstanceVersion) completed end to end correctly over TCP at least once** —
concrete "yes" evidence for the question "can Venus/vtest work over a WAN transport".

The remaining problem is no longer an architectural blocker but **a pure
synchronization-timing problem**: for the reply pool, instead of "blindly poll
periodically", we need "do a synchronous GET at EXACTLY the moment we know the host has
written the reply". This can be tied to a guarantee the ring already provides: the host
does not advance the ring seqno BEFORE writing the reply (this ordering guarantee exists in
the Venus protocol itself) — so the moment `vn_ring_wait_seqno()` returns successfully, the
host's reply must ALREADY be written. The only problem is that our TCP sync mechanism does
not catch this FAST ENOUGH.

## Concrete Task for the Next Round

1. Find the successful return point of `vn_ring_wait_seqno()` in `vn_ring.c` (when the seqno target is reached).
2. There (or at the point where the call to `vn_ring_submit_command()` returns), add a hook that triggers **a synchronous, mandatory GET** for the pending reply-pool resource, if any — instead of relying on the background poll thread. This may require a call specific to `vn_renderer_vtest.c` from inside `vn_ring.c` (renderer-agnostic); for this experimental branch, adding a pragmatic function pointer/hook instead of architectural purity is acceptable.
3. Alternative (simpler, untried): find a common point (if any) that the `vn_call_*` functions call right before reading the reply, and hook there.
4. After the fix, run `vulkaninfo --summary` at least 5 times in a row and verify that **all of them succeed** (a ~50% success rate is currently observed; a consistent 100% is the goal).
5. If successful, mark phase 3 COMPLETE: update the plan file, save the full output where `vulkaninfo` shows `driverName: venus` (similar to phase 2's `05-SUCCESS-end-to-end.md`), then move on to the `tc netem` artificial delay test.

## Commands used (reference)

Server:
```
VIRGL_LOG_LEVEL=debug ./vendor/virglrenderer/build/vtest/virgl_test_server \
  --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path tcp:33445
```

Client:
```
ICD=$(find vendor/mesa/build -name "virtio_devenv_icd.x86_64.json")
VK_ICD_FILENAMES="$ICD" VTEST_SOCKET_NAME=tcp:127.0.0.1:33445 vulkaninfo --summary
```

## Note: temporary debug instrumentation

The following temporary debug `fprintf`s were added this round (can be cleaned up later, useful for now):
- `vtest_renderer.c`: a dispatch-failure print in `vtest_client_dispatch_commands`, a reply-pool byte dump in `transfer_get_internal`.
- `vn_renderer_vtest.c`: a sync-thread start print.
- `vn_protocol_driver_instance.h` (meson-generated, under build/ — **lost on reconfigure/clean build**, not permanent): a raw byte dump before the assertion in the reply decode.

Without `VIRGL_LOG_LEVEL=debug`, `vkr_log()` calls (the actual Venus error messages on the
host side) are **silently suppressed** (the default log level is WARNING) — this explains
why the CS error was not visible at the start of this round. This environment variable must
be used in all server tests from now on.
