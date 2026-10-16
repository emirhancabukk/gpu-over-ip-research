# Phase 3 — First TCP Transport Attempt: The Phase 1 Prediction Confirmed (2026-08-27, autonomous round 7)

## The change made

A backward-compatible change was added to both the client
(`vendor/mesa/src/virtio/vulkan/vn_renderer_vtest.c`) and the host
(`vendor/virglrenderer/vtest/vtest_server.c`) that **also supports a TCP socket with a
"tcp:" prefix**, without breaking the existing Unix-socket behaviour:

- Client: `VTEST_SOCKET_NAME=tcp:<host>:<port>` → `vtest_connect_socket_tcp()` (AF_INET connect, TCP_NODELAY on)
- Server: `--socket-path tcp:<port>` → `vtest_server_open_socket_tcp()` (AF_INET bind INADDR_ANY, listen)

Both were built and tested on localhost (`tcp:127.0.0.1:15150` ↔ `--socket-path tcp:15150`).

## Result: PARTIAL SUCCESS + the phase 1 prediction confirmed exactly

- Vulkan **context creation also succeeds over TCP** (`vtest_lazy_init_context: ret=0` — the same success as in phase 2; the transport makes no difference).
- But the client crashed (SIGABRT). A full stack trace was taken with `gdb -batch -ex run -ex bt`:

```
#3  vtest_receive_fd (...) at vn_renderer_vtest.c:197   <-- abort() here
#4  vtest_vcmd_resource_create_blob (...) at vn_renderer_vtest.c:404
#5  vtest_shmem_create (...) at vn_renderer_vtest.c:838
#7  vn_renderer_shmem_create (...)
#8  vn_ring_create (...) at vn_ring.c:303
#9  vn_instance_init_ring (...)
#10 vn_CreateInstance (...)
```

The source of `vtest_receive_fd()` (lines 172-201): it reads **`SCM_RIGHTS`** ancillary
data with `recvmsg()` — the classic Unix-domain-socket **file descriptor passing**
mechanism. If the `cmsg` is not valid/SCM_RIGHTS, it calls `abort()` (lines 196-198),
exactly the crash we saw.

## This fully confirms what we predicted in phase 1

In `docs/research-notes/phase1-source-code-findings.md` we wrote: *"vtest's resource
creation side (vtest_new_shm) uses memfd + fd passing, which does not move to a network."*
It is now **proven** with a real run and a full stack trace: `vn_ring_create` (the first
thing Venus does to set up its guest↔host command ring) immediately calls
`vn_renderer_shmem_create`, which goes `vtest_shmem_create` →
`vtest_vcmd_resource_create_blob` → **tries to receive a shared-memory fd from the server
via SCM_RIGHTS**. SCM_RIGHTS ancillary data cannot travel over a TCP socket (an OS/kernel
restriction; it is a feature specific to Unix domain sockets), the server probably fails to
send it silently or does not send it at all, and the client receives an invalid/empty cmsg
and aborts.

## What it means: the real difficulty of phase 3 is now clear

The simple "change the socket type" approach is **not enough**, because even the setup of
Venus's command ring itself depends on shmem + fd passing — this is not only for large
textures/buffers but at the very foundation of the protocol (`vn_instance_init_ring`, that
is, right at the start of instance creation). So to make a WAN transport work, either:

1. **`vtest_shmem_create` has to be redirected to a different mechanism in TCP mode** — instead of fd passing, use plain memory `malloc`ed on the server and write a "TCP shmem emulation" that synchronizes the ring buffer contents periodically with explicit `transfer_get/put` commands (the phase 1 finding that transfer_get/put is a byte stream will be useful here). This means the ring is no longer "real shared memory" but "a local copy synchronized periodically" — there is a performance impact, but it works over a WAN.
2. Or: set up an extra side channel over the TCP connection (e.g. create a Unix socket pair on the first connection and use it only when fd passing is needed) — but this requires the host and the client to be on the same machine, so it is meaningless in a WAN scenario.

**Conclusion:** path 1 (shmem emulation / synchronizing the ring through transfer_get/put)
is the only realistic option. This is a bigger engineering job than we first thought —
Venus's ring mechanism (`vn_ring_create`/`vn_renderer_shmem_create` on the guest side, the
counterpart of `vtest_shmem_create` on the host side) has to be replaced with a TCP-compatible
shmem emulation layer.

## Concrete Task for the Next Round
1. Read the FULL code of `vtest_vcmd_resource_create_blob` (`vn_renderer_vtest.c` lines ~380-410) and also its server-side counterpart (the resource_create_blob handler in `vtest_renderer.c`) — pin down exactly where/how the fd passing happens and what the blob resource is used for (only for the ring, or also for textures/buffers).
2. Draft a simple "TCP shmem emulation" design: keep the blob in real memory on the server side, and let the client synchronize periodically with `transfer_get/put` (or a new dedicated command). Think about how this affects the ring buffer semantics (head/tail pointer polling) — this is directly connected to the phase 1 finding that "the ring is built on real shared memory + polling"; both come down to the same root.
