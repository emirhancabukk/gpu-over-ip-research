# Phase 1 - Prior Art Review Findings (2026-08-27)

## The Mesa Venus Protocol

**Architectural layers:**
- Guest side: the Venus Vulkan driver (appears to the application as a Vulkan ICD)
- Transport: the `virtio-gpu` driver in the guest kernel — 5 specific parameters are required to talk to the host: `VIRTGPU_PARAM_3D_FEATURES`, `VIRTGPU_PARAM_CAPSET_QUERY_FIX`, `VIRTGPU_PARAM_RESOURCE_BLOB`, `VIRTGPU_PARAM_HOST_VISIBLE`, `VIRTGPU_PARAM_CONTEXT_INIT`
- Host side: **virglrenderer** — requires Vulkan 1.1+ and platform-specific external memory extensions

**IMPORTANT FINDING — commands are sent in batches per command buffer, not per draw call:**
Venus accumulates all commands until `vkEndCommandBuffer` is called and sends them to the
host **in one go**. So the round-trip granularity is already not "a separate network trip
for every draw call" but "one network trip per command buffer". The biggest risk in our
plan (thousands of draw calls = thousands of round trips) is largely already solved by
Venus. This makes moving to a WAN transport more favourable than we hoped.

**Caching:** queries such as `vkGetPhysicalDeviceFeatures2()` are already fetched and
cached on the first `vkEnumeratePhysicalDevices()`/`vkEnumeratePhysicalDeviceGroups()`
call, and do not go back to the host. The general principle: static/unchanging
information is kept in a local cache and unnecessary round trips are eliminated.

**Missing/unclear (not in the documentation, must be read from the source code):**
- The exact mechanics of fence/semaphore/timeline synchronization are not explained — only future `virgl_renderer_export_fence` support is mentioned. This is **the part to study in depth in phases 2/3**, because synchronous wait points are where a WAN hurts most.
- No performance/latency figures are published at all.

**Risk note:** Venus makes implementation-specific assumptions for host-visible
`VkDeviceMemory` that violate the spec (`vkMapMemory` behaviour). When changing the
transport, these assumptions must not be broken — noted as a point to watch while changing
the architecture.

## RVGPU (Panasonic Automotive) — CORRECTION

Contrary to the plan's initial assumption, RVGPU is the **exact opposite** of the
architecture we want: `rvgpu-proxy` (the client) does **local rendering** on the machine
with the GPU, and `rvgpu-renderer` (the server) on a separate machine only **receives and
displays the image** (over TCP, e.g. `127.0.0.1:55667`). So this is not "remote GPU,
display the result locally" but the classic "render locally, watch remotely" (closer to
VNC/streaming).

**Usefulness:** it does not fit us as an architectural template, but its TCP transport
implementation (librvgpu) may serve as a code-level reference for writing a network
transport layer. We will evaluate that part separately and no longer count it as an
overall architectural model.

## Conclusion / Input to Phase 2

Venus + virglrenderer is a better foundation than we first thought, because it already
batches at the command-buffer level. The real unknown: **how fence/semaphore
synchronization will behave under WAN latency**. This is not in the documentation and has
to be read directly from the virglrenderer source code (the `src/venus/` directory). We
will study it at the source-code level as part of phase 2 (local PoC).
