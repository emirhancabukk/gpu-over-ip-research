# Phase 2 — Root Cause of CONTEXT_FAILED Found (2026-08-27, autonomous round 6)

## The previous round's hypothesis was WRONG, but debugging corrected it quickly

The previous round guessed that "the renderer name string may be partially read". When a
temporary debug line was added to `vtest_create_context()` and tested: `expected length=11
read_ret=11` — **an exact match, this function returns successfully.** Hypothesis ruled out.

## The real root cause: `proxy_context_create()` returns NULL → ENOMEM

A second debug line was added to `vtest_lazy_init_context()` and tested again:
```
[DEBUG] vtest_lazy_init_context: capset_id=4 ret=12 errno=... (noise, ignored)
```
`capset_id=4` = `VIRTGPU_DRM_CAPSET_VENUS`. `ret=12` = **ENOMEM** (errno 12 on Linux). Looking inside `src/virglrenderer.c:246-268` (`virgl_renderer_context_create_with_flags`):

```c
case VIRTGPU_DRM_CAPSET_VENUS:
   if (!state.proxy_initialized)
      return EINVAL;
   ctx = proxy_context_create(ctx_id, ctx_flags, nlen, name);
   break;
...
if (!ctx)
   return ENOMEM;   // <-- this is where we end up
```

So **`proxy_context_create()` returns NULL.** Judging by the `render server mode:
process` / `render server worker: process` settings we saw in the meson build output,
virglrenderer's Venus support hands the real Vulkan work to **a separate child process (the
render server)** for security reasons — the name "proxy" confirms this too.
`proxy_context_create()` returning NULL probably indicates that **this child process could
not be started / fork+exec failed** (possibly because of sandboxing, permissions, or a
restriction in this environment).

## Temporary changes made (inside the project, no effect on the system)
Two temporary `fprintf(stderr, ...)` debug lines were added to
`vendor/virglrenderer/vtest/vtest_renderer.c` (visible with git diff; nothing to send
upstream, for our debugging only). They can be reverted with `git checkout` if wanted, but
the next round will probably need them, so they were left in for now.

## Concrete Task for the Next Round
Find the `proxy_context_create()` function (probably `src/proxy/proxy_context.c` or
similar; we saw `proxy_proxy_context.c.o` in the meson output) and read the conditions under
which it returns NULL — probably at a point where a `fork()`/`exec()` or `minijail` call
fails. If spawning the render server subprocess does not work in this environment (e.g.
because of a sandbox/container restriction), try reconfiguring with the
`--render-server-worker` build option set to `thread` instead of `process` (this option was
already seen in meson_options.txt) — this uses a thread inside the same process instead of a
child process and may bypass the subprocess spawn restriction.
