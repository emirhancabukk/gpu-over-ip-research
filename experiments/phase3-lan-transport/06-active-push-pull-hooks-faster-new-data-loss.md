# Phase 3 — Active Push/Pull Hooks Increased Speed; A Clear "Trailing Zero Bytes" Finding (2026-08-28, autonomous round 11)

## Done this round

Both concrete tasks from note 05 were implemented:

1. **A host-side debug dump** (`vendor/virglrenderer/src/venus/vkr_ring.c`, `vkr_ring_submit_cmd()`): when the decoder becomes fatal, it logs the first 64 bytes of the buffer being decoded + the `dec->cur` offset in hex. This produced the most valuable finding of this round (below).

2. **Active push/pull for the ring** (`vn_ring.c`, `vn_renderer_vtest.c`, `vn_renderer.h`): the `sync_now` hook, added in the previous round ONLY for the reply pool, is now also used for the ring itself:
   - **Push:** in `vn_ring_submit_internal()`, right after `vn_ring_store_tail()`, it sends the newly written command data to the host IMMEDIATELY, without waiting for the background thread's next ~200µs pass.
   - **Pull:** on EVERY iteration of `vn_ring_wait_seqno()`'s wait loop, it actively pulls the host's progress (instead of relying only on the background thread's own pace).
   - Both go through `vtest_shmem_sync_now()` and call the ring-specific `vtest_shmem_sync_ring_cycle()` function (already ownership-respecting and correctly ordered) — no blind full-buffer GET.

## Result: a visible speed increase, but a NEW (or previously rare) data problem appeared

After this change, the first run **got further than we had ever reached** (the ring head
advanced to 2308 — dozens of commands processed successfully), but in the end it crashed
again with `vkCreateInstance resulted in CS error`. Thanks to the newly added host-side
debug dump, this time **clear evidence** was obtained:

```
ring_submit_cmd: fatal at dec->cur offset=32/36 size=36 ring_head=2308
first_36_bytes: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
                00 00 00 00
```

**Interpretation:** the decoder **successfully decoded 32 bytes** of a 36-byte command; only
**the last 4 bytes** are a problem — and those last 4 bytes are **entirely zero**. This does
not look like random corruption (a bit flip, a wrong offset); on the contrary, it is a clean
zero-fill pattern carrying the signature "this memory location has never been written / never
reached the host". This is NOW a specific and testable hypothesis: **at the moment the
client's push announces the tail to the host, the corresponding buffer bytes may not yet be
FULLY reflected in the host's memory** — the "buffer first, tail after" fix from note 03
guarantees the order of the PUTs AMONG THEMSELVES, but it may not completely close the host
ring consumer thread's RACE WINDOW (e.g. the host's own memory write/synchronization
behaviour, or our active push/pull hooks NOW firing much more often/faster may expose a
previously rare window more often).

## Concrete Task for the Next Round

1. **Add a matching debug print on the client side too**: on every `vtest_shmem_sync_ring_cycle()` call, log the tail value that is PUT and a checksum of the pushed region of the buffer (e.g. a hash of the last few hundred written bytes, or just the last N bytes). Compare it with the host's dump at the fatal moment — is what the client sends REALLY different from what the host receives, or is the client already sending WRONG/incomplete data (a push may have been triggered before the local write completed)?
2. **Hypothesis A (client-side race):** check how `vn_ring_write_buffer()` itself does the write — if there are several `cs->buffers[i]` (a multi-part command) and at ANY intermediate step (e.g. during a wait inside `vn_ring_wait_space`) our NEWLY added active pull/push hooks (especially `vn_ring_wait_seqno`'s pull, IF `vn_ring_wait_space` also triggers it indirectly) could trigger a push IN BETWEEN and send an INCOMPLETE write to the host — check this.
3. **Hypothesis B (host-side visibility):** the host's `vkr_ring_thread` reads `ring->control.tail` with `memory_order_acquire` (correct), but there is NO synchronization primitive when it reads the DATA bytes (`buf->data`, a separate memory region) — it relies only on the assumption "if the tail increased, the data must be there too". In our TCP-sync model, PUT-buffer and PUT-tail are SEPARATE VTEST COMMANDS (separate `vtest_block_write` calls, possibly separate TCP segments) — **even if the host's server dispatch thread processes them in order**, there is no REAL memory barrier/atomic synchronization between the host's `memcpy` into the `res->iov.iov_base` memory and the reads of the ring consumer thread (a DIFFERENT thread) (only `tail` is atomic, the `buffer` region is NOT). Although normal stores become visible in order on x86, there is NO synchronization primitive (mutex/atomic/fence) between the two SEPARATE threads (the dispatch thread's memcpy vs the ring consumer thread's read) — in theory the data-visibility guarantee may be missing. To verify: examine whether, on the server side, an `atomic_thread_fence` right after the PUT-buffer is processed, or a mechanism that synchronizes with the host's ring consumer thread before the PUT-tail is processed, has been added.
4. If B is right, the fix: adding an `atomic_thread_fence(memory_order_release)` at the end of the server's PUT-buffer handler (`vtest_transfer_put_internal`, right after the memcpy) may be enough, assuming the ring consumer's tail read already pairs with it via `memory_order_acquire` (for release-acquire pairing, the side the server WRITES must also carry release semantics).
5. If found, fix it, rebuild, and retry the consistent 5/5 `vulkaninfo --summary` test.

## Note
The debug logs are now quite crowded (high-frequency prints such as `server transfer_get on
reply-pool res`) — they can be throttled with an env gate later if needed; left untouched for now.
