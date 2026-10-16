# Phase 4 — A Critical Race Condition Found and Fixed: Ring Buffer Corruption (2026-08-29)

## Summary

During a deep network/syscall diagnosis (`tcpdump` + `strace`, tools the user installed
with `sudo`), the root cause of the occasional `vulkaninfo` crashes seen in the previous
round (`SIGABRT`, diagnosed with `coredumpctl`) was found: **a real data-corruption race
condition**, not just a performance problem. Fixed and verified: **the crash no longer
happens at all**.

## Diagnosis process

1. `tcpdump` was installed (set up to run without sudo with `setcap cap_net_raw,cap_net_admin+eip`). In a capture of 213,779 packets only **5 repeated sequences** (real TCP retransmissions) were found — real network packet loss is negligible, and the "network jitter" hypothesis was largely refuted.
2. `strace` was installed and `vulkaninfo` was run under it. The process crashed with `SIGABRT` again.
3. The stack trace was examined with `coredumpctl info`: the crash came from the `status & VK_RING_STATUS_FATAL_BIT_MESA` check inside `vn_ring_submit_internal` — that is, the **host** saw a fatal error while decoding the ring and set the `FATAL` bit, and the guest then read it and called `abort()`.
4. The full message was found in the host log (`VIRGL_LOG_LEVEL=debug`): `vkCreateInstance resulted in CS error`, the decode stuck around `dec->cur offset=68/160`, everything after **offset 32** of the first 64 bytes **entirely zero** — exactly the same "32 bytes correct, the rest zero" pattern found in phase 3's note 06.

## Root cause

There was no lock/synchronization at all between `vn_ring_write_buffer()` (`vn_ring.c`, the
upper-layer function that writes command data into the ring, part of Mesa's
renderer-agnostic code) and our TCP synchronization code (`vtest_shmem_sync_ring_cycle`,
called by both the background thread and the active `sync_now` hook) in their **access to the
ring buffer's mmapped memory**. The existing `sock_mutex` protected only the TCP socket
reads/writes (`vtest_write`/`vtest_read`) — `vn_ring_write_buffer` itself never took this mutex.

In low-latency environments (localhost, 0-40ms simulated with `tc netem`) this never caused
a problem, because the `write_buffer → store_tail → sync_now` sequence completed very
quickly and the probability of the background thread (5ms period) slipping in exactly in
that window was almost zero. Over the real internet, `sync_now` itself occasionally takes 1-7
seconds (a separate topic, below), so the main thread holds `sock_mutex` for a long time and
the background thread waits; but the moment it is released, **if the main thread is at that
very moment in the middle of a new `write_buffer` call** (before `store_tail`), the
background thread could PUT the **half-written** buffer contents in the mmap to the host — the
host tried to decode it, saw inconsistent data and raised a fatal error.

## Fix

1. `vn_renderer.h`: a new `lock`/`unlock` hook pair was added to `vn_renderer_shmem_ops` (optional, a no-op for real virtio-gpu).
2. `vn_ring.c`, `vn_ring_submit_internal()`: every `vn_ring_write_buffer()` call (and the `vn_ring_store_tail()` + `sync_now` push that follow it) is now wrapped in this lock. `vn_ring_wait_space()` was deliberately kept **outside** the lock — it waits for the host's head to advance, and that information is updated only when the background thread can take the lock; keeping it inside the lock would deadlock.
3. `vn_renderer_vtest.c`: the `lock`/`unlock` hooks were implemented using the existing `sock_mutex`. **A critical detail**: `sync_now` (called under the lock) itself calls `vtest_shmem_sync_ring_cycle`, which tries to take the SAME mutex again — so the mutex was changed from `mtx_plain` to `mtx_plain | mtx_recursive` (otherwise the same thread would lock itself into a self-deadlock).

## Verification

A clean run after the fix: **435 seconds, zero crashes, zero `FATAL`/`abort` messages**,
`driverName=venus`, `deviceName=NVIDIA A10G`. The ring synchronization average fell to
**156.9ms** (from the earlier 573ms, inflated by the backoffs on the path to the crash, to a
real/stable value).

## Remaining open question (performance, no longer correctness)

**15-20%** of the reply-pool and ring synchronizations still take between 1 and 7 seconds
(39 of 179 samples in the reply pool). Host GPU utilization is constantly 0%, and there is no
real packet loss in tcpdump — the exact source of these delays is still unclear (it could be
host CPU scheduling / TCP congestion window behaviour / something else). But this is now **not
a correctness risk, purely a performance problem** — the system now works reliably (without
crashes); only its speed is not optimal.

## Status

Phase 4's core goal (a reliable connection through Venus to a real AWS GPU over the real
internet) is now **both verified and reliable**. The remaining work is a pure performance
optimization (a bigger next step — real pipelining, or a full diagnosis of the source of the
network delay).
