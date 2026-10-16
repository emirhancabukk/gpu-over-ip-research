# Phase 3 — "Phantom Byte" Bug Found and Fixed, Now a Timing Problem (2026-08-27, autonomous round 8)

## Summary of this round: 4 real bugs, 3 of them fixed

In this session, 4 different, real bugs were found one after another in the TCP transport attempt, and most were fixed — in order:

1. ~~The SCM_RIGHTS crash~~ → fixed (previous round, local anonymous mmap + sync thread)
2. ~~TRANSFER_GET2/PUT2 not reading data_size~~ → fixed (previous round, switched to the old GET/PUT opcodes)
3. ~~transfer_get/put not working on blob resources~~ → fixed (this round: `res->iov` is now also filled for HOST3D blobs in `vtest_resource_create_blob`, and a direct memcpy fast path for blobs was added in transfer_get/put_internal)
4. **The newly found "phantom byte" bug** → fixed

## Bug #4: `vtest_send_fd()` silently sends 1 extra byte over TCP

`vtest_send_fd()` (server, `vtest_renderer.c:439`) sends a 1-byte dummy payload + SCM_RIGHTS
ancillary data with `sendmsg()`. **On an AF_INET/TCP socket the kernel silently ignores the
SCM_RIGHTS ancillary data, but the 1-byte dummy payload is STILL sent** (`sendmsg` does not
return an error, `size>=0`). The client in TCP mode already skips receiving the fd (the fix
for bug #1), so it **never reads** this extra byte — it stays in the stream and shifts the
start of the next command's header by 1 byte. This is a cumulative stream corruption that
repeats on every `vtest_send_fd()` call (blob create, sync create, fence export, etc. —
there are ~10 call sites in the code base).

**Chain of evidence:** no problem showed for the first blob (the ring, size 131268) —
by coincidence, the next command's byte alignment happened to tolerate it — but when the
SECOND blob (the reply shmem, size 1048576) was created, the client's response-header
assertion (`vtest_hdr[VTEST_CMD_LEN]==1`) fired. Temporarily turning off the sync thread
with the `VTEST_TCP_NO_SYNC_THREAD=1` environment variable and seeing the same error AGAIN
(see the temporary env gate in `vendor/mesa/.../vn_renderer_vtest.c`) made it certain that
this was **not a threading race but a deterministic framing bug** — a good, time-saving
diagnostic step.

**Fix:** a check that inspects the socket family with `getsockname()` was added to
`vtest_send_fd()` — if it is not `AF_UNIX` (that is, if it is TCP), the function returns
`0` (success) without sending anything.

## State after the fix: the framing bug is gone, now there is a TIMING problem

After the phantom-byte fix:
- With the sync thread off: `vn_ring_wait_seqno` waits forever and aborts through `vn_relax` (**expected/correct behaviour** — without sync the ring does not advance, an outcome our design predicts).
- With the sync thread on (2 separate sync threads run for the 2 blobs, doing a `transfer_put`/`get` pair every 2ms): it **still** gets stuck in the same place (`vn_ring_wait_seqno` → `vn_relax` → abort), but it is no longer a framing error — probably the **speed** of the sync threads' synchronization is too slow relative to `vn_relax`'s fixed retry count (`vn_relax` seems to give up based on the iteration count, not on wall-clock time — our TCP round trips exceed it).

## Concrete Task for the Next Round
1. Read the `vn_relax()` function in `vn_common.c` — pin down after how many iterations/how much time it gives up.
2. Try removing (or greatly shortening) the sync thread's 2ms `nanosleep` and polling as fast as possible — at least to prove "it works", without worrying about performance (that is phase 5's job).
3. Alternative: for a test, make the polling of ONLY ONE sync thread (the ring's) very aggressive (spin loop, no sleep) and test in isolation, leaving the reply-shmem thread separate — this shows which ring's latency is critical.
4. If aggressive polling is not enough either, consider increasing `vn_relax`'s retry limit at build time (experimental, only for our TCP prototype).

## Note: temporary debug code behind an env gate
The `VTEST_TCP_NO_SYNC_THREAD` environment variable was added to
`vendor/mesa/.../vn_renderer_vtest.c` as a temporary diagnostic tool — not a permanent
feature; it can be cleaned up later, but it was left in for now because it is useful for
quick isolated tests.
