# Phase 1 — vtest_renderer.c: the transfer_get/put and sync mechanism (Autonomous loop, round 2)

## Finding — vtest's transfer/sync commands ARE network-friendly; they do not use memfd/fd passing

Read `vtest_transfer_get_internal`/`vtest_transfer_put_internal` (lines ~1568-1671) and
`vtest_sync_wait` (line ~2152) in `vtest/vtest_renderer.c`:

- **`vtest_transfer_put`**: reads the data with `ctx->input->read(ctx->input, data_iov.iov_base, ...)` — a **plain byte stream** over the socket/fd. It then writes it to the GPU resource with `virgl_renderer_transfer_write_iov()`.
- **`vtest_transfer_get`**: reads from the GPU resource with `virgl_renderer_transfer_read_iov()` and sends it back over the socket as a **plain byte stream** with `vtest_block_write(ctx->out_fd, ...)`.
- **`vtest_sync_wait`**: sync IDs, flags and timeout values are all read as **message/command data** with `ctx->input->read()` — no shared memory, no fd passing.

**Conclusion:** the concern from the previous round — "vtest's resource creation side
(`vtest_new_shm`) uses memfd + fd passing, which does not move to a network" — still holds,
BUT only for **large/persistent resources** (textures, buffers). **The transfer_get/put
and sync_wait commands themselves are already a complete byte-stream protocol** —
connecting them to a TCP/QUIC socket instead of a Unix socket needs no extra translation
layer for these commands; changing the underlying socket implementation is enough.

**Practical implication:** two resource-management strategies are possible for a WAN prototype:
1. Simple but slower: do all resource data transfer with `transfer_get/put` commands (never use memfd/shared memory) — this may be slow (an explicit copy on every access) but **works over TCP right away**, ideal for a first prototype.
2. Faster but more complex: keep the memfd/shared-memory fast path on the local machine and fall back to the transfer_get/put path only when the host is actually remote (when a network transport is detected) — a hybrid.

Starting the phase 3 prototype with **strategy 1** makes sense — simple, works, optimize later.

## Obstacle hit this round: the local build attempt needs sudo

I tried to install `meson`/`ninja` and the build dependencies (`libepoxy`, `libdrm`,
`mesa`, `vulkan-headers` etc.) with `sudo pacman -S`, but in this environment **a
non-interactive (autonomous) sudo password prompt failed** ("a password is required", no
terminal). This is not a security/architectural restriction on installing packages on the
main system itself (the plan already allowed this as "ordinary development packages") —
the autonomous loop simply cannot enter a password.

**Needed from the user:** to move this step forward, the user needs to run this command
once themselves (or define passwordless sudo for pacman so the loop can install it itself,
but I do not recommend that — the security choice belongs to the user):

```
sudo pacman -S --needed meson ninja pkgconf libepoxy libdrm mesa vulkan-headers vulkan-icd-loader wayland-protocols
```

Until this is run, the loop will continue with source-code-reading research other than the build attempt.
