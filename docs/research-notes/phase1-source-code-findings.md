# Phase 1/2 — virglrenderer Source Code Review (Autonomous loop, round 1)

## Done
- Cloned the real repo into `~/Desktop/gpu-over-ip-research/vendor/virglrenderer/`
  (gitlab.freedesktop.org could not be read with WebFetch because of the Anubis anti-bot
  protection on its HTTP pages, but `git clone` worked at the protocol level — noting
  this: if we need to fetch something over HTTP again later, try `git clone`/`curl`, not
  a browser fetch).
- Checked the local machine's KVM/virtualization support: **AMD-V is present, `/dev/kvm`
  exists, the `kvm_amd` module is already loaded.** So a VM-based approach (for phases
  2/3) is technically possible on this machine. `qemu` is not installed yet.
- Read `src/venus/vkr_transport.c`, `src/venus/vkr_ring.c`, `vtest/vtest_shm.c`, `vtest/vtest.h`.

## CRITICAL FINDING 1 — Venus's ring buffer is built on real shared memory and cannot be moved to a network (as-is)

The mechanism in `vkr_ring.c` works like this:
- The guest and the host share **the same physical memory region** (a virtio-gpu blob resource of type `VIRGL_RESOURCE_FD_SHM`).
- On the host side, a separate thread (`vkr_ring_thread`) continuously reads the `head`/`tail` pointers in this shared memory (`atomic_load_explicit`) and **polls** for new commands (busy-wait → sleep with exponential backoff, `vkr_ring_relax`).
- This is not the guest "sending" commands to the host; it is the guest writing to memory and the host watching the same memory — **it requires being physically on the same machine.**

Conclusion: we cannot move Venus's ring transport layer to a WAN just by saying "carry it
over a socket" — this is a harder problem than the first draft of the plan assumed. We
need to turn this shared-memory/polling model into a real message-passing model (notify
the host "there is a new command" + send the data over the socket).

## CRITICAL FINDING 2 — vtest's resource sharing also uses memfd + fd passing, which is also local-only

`vtest_shm.c`: `vtest_new_shm()` → creates a memory-backed file descriptor (fd) with
`memfd_create()`. Sharing this fd between client↔server must be done with **SCM_RIGHTS**
(fd passing) over a Unix domain socket (the code is not in this file, but memfd_create +
a Unix socket is the classic SCM_RIGHTS pattern). **This does not work over a network
(TCP/QUIC) either** — file descriptors are valid only within OS boundaries; you cannot
"send an fd" over a network socket, only bytes.

So: replacing the Unix socket with TCP is not enough — resource data transfer
(texture/buffer contents) has to be turned from fd passing into real byte-stream
serialization.

## PROMISING FINDING — the vtest protocol already has message-based sync and transfer commands

`vtest.h` defines the following functions:
- `vtest_transfer_get` / `vtest_transfer_put` (and their `_get2`/`_put2` variants) — instead of shared memory, these are **explicit data-copy commands**; as far as their names suggest, they carry the data as protocol messages. This could be a natural integration point for our WAN transport — a path built on these transfer commands instead of the ring/shm.
- `vtest_sync_create/unref/read/write/wait`, `vtest_drm_sync_*` (including timeline signal/wait) — synchronization is also defined here at the message/command level, instead of raw shared memory.

**Suggested next step:** read the implementations of `vtest_transfer_get/put` and
`vtest_sync_*` in `vtest_renderer.c` (81KB, not read yet) and verify whether they are
truly network-friendly (a full byte stream, no fd passing). If so, the goal of phase 3
becomes clear: make **vtest's transfer/sync command protocol**, not Venus's SHM ring, the
foundation of the WAN transport.

## Plan Update (proposal, not yet written into the plan)
The phase 2/3 strategy could be updated: instead of setting up a full crosvm/QEMU VM,
first bring up the **vtest_server** (host) + Venus/Mesa Vulkan client (guest side; it can
also run without a VM, as the Venus ICD) pair over a local Unix socket and verify it, then
replace the Unix socket with TCP + explicit resource serialization. This has far fewer
moving parts than a full VM. Before writing this change into the plan, I will read
vtest_renderer.c and confirm the mechanism really fits.

## Concrete Tasks for the Next Round
1. Read the implementation of the transfer_get/put and sync functions in `vtest_renderer.c`.
2. If they are network-friendly, update the phase 2/3 description in the plan file (a vtest-based approach instead of crosvm/QEMU).
3. Install `meson`/`ninja` (ordinary development packages, no risky lasting effect on the main system) and try building virglrenderer with `-Dvenus=true` — the local AMD APU already has a Vulkan/RADV driver, so we can test the host side locally (no AWS, free).
