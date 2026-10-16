# Phase 3 — transfer_get/put Does Not Support Blob Resources (2026-08-27, autonomous round 7 continued)

## Fixes made this round (in order)

1. **The SCM_RIGHTS crash was fixed**: in TCP mode (`vtest->is_tcp`), `vtest_vcmd_resource_create_blob` no longer calls `vtest_receive_fd()` and returns `res_fd=-1`. If `res_fd<0`, `vtest_shmem_create` uses a **local anonymous mmap** (`MAP_ANONYMOUS`) instead of a real fd and starts a **synchronization thread** in the background (`vtest_shmem_sync_thread_func`) — this thread periodically (every 2ms) sends the local memory to the host with `transfer_put` and pulls it back with `transfer_get`.

2. **First bug**: I used `VCMD_TRANSFER_GET2`/`PUT2` (opcodes 13/14) — on the server these **never read the data_size field** (the `DATA_SIZE` field is not parsed in `vtest_transfer_decode_args2`), they are always called with `data_size=0`, so they **transfer 0 bytes.** While my client expected to write/read `size` bytes, the server sent/read 0 bytes and the stream broke (the next command's header started being read from the wrong place — `Assertion vtest_hdr[VTEST_CMD_LEN]==1 failed`, `VTEST_CLIENT_ERROR_INPUT_READ`).

3. **Fix**: switched to the older `VCMD_TRANSFER_GET`/`PUT` commands (opcodes 4/5, an 11-dword header, the `DATA_SIZE` field really is read — `*data_size = thdr_buf[VCMD_TRANSFER_DATA_SIZE]` in `vtest_transfer_decode_args`). The stream corruption is fixed; we now get **a clean protocol-level error**.

## A new (and probably fundamental) obstacle: transfer_get/put does not work on blob-type resources

```
vtest_transfer_put_internal called virgl_renderer_transfer_write_iov which failed (-1)
client: VTEST_CLIENT_ERROR_COMMAND_DISPATCH
```

Read the source of `virgl_renderer_transfer_write_iov()` (`src/virglrenderer.c:316`): it
forwards the call to the context's `transfer_3d()` callback (`ctx->transfer_3d(ctx, res,
&transfer_info, VIRGL_TRANSFER_TO_HOST)`). Our context is a **Venus proxy context**
(capset_id=4) — this proxy's `transfer_3d` implementation **fails for blob-type resources.**

**Plausible explanation:** blob resources (host-visible `VkDeviceMemory` / the ring buffer
memory) are by design meant to be **accessed directly through mmap** — the "transfer"
(box-copy) API exists for classic virgl 2D/3D texture resources (with level/format/stride).
There is **probably no way at all** to access a blob's bytes with the box-copy API — a
natural consequence of the design the vtest protocol is built on, not of us misusing
something.

## Conclusion / Overall Assessment

The finding we noted hopefully in phase 1 — "transfer_get/put is already a byte stream,
suitable for a WAN" — **turned out to be partly wrong**: transfer_get/put does mechanically
carry a byte stream (correct), but **only for classic texture resources**; it **does not
work** for blob resources (exactly what Venus needs). This means **there is no ready-made
solution** in the current vtest protocol — for blob synchronization over a WAN, either:

1. **A new command has to be added to the vtest protocol** — a simple "blob_get/blob_put" command, with no notion of box/format, that does a direct memcpy into the blob's backing memory on the server side (probably `res->iov` or the proxy context's own memory area). This means writing a real change to virglrenderer's server side (vtest_renderer.c + vtest_protocol.h + the proxy context code that handles the protocol) — a small but real protocol extension.
2. Or: find out how the blob is stored inside the proxy context (in the render server worker process/thread) and research whether there is a different API/function that provides direct memory access to it on the host side (e.g. there may be a function in `vkr_renderer.c`/`proxy_context.c` that "maps" the blob; expose it with a new vtest command).

## Concrete Task for the Next Round
1. Find how blob resources are represented on the host side in `vendor/virglrenderer/src/proxy/` (or the `proxy_proxy_*.c` files we saw in the meson output) — probably stored as a `void *` pointer + size (because it had to be mmap-able).
2. Check whether there is (or could be added) a function that gives access to that pointer.
3. If found, add a new simple command to `vtest_protocol.h` (e.g. `VCMD_BLOB_READ`/`VCMD_BLOB_WRITE`, only res_id + offset + size + data) and write a minimal implementation in both `vtest_server.c`/`vtest_renderer.c` (host) and `vn_renderer_vtest.c` (client). This is writing a real protocol extension — a big step, but a clear goal.
