# Phase 2 — Mesa virtio Vulkan ICD Built, End-to-End Attempt (2026-08-27, autonomous round 4)

## Done

1. Cloned the Mesa source code (`vendor/mesa`, shallow clone, 546MB).
2. Built only the virtio Vulkan driver with Meson (OpenGL/EGL/Gallium disabled, to shorten the build):
   ```
   source vendor/.build-venv/bin/activate
   cd vendor/mesa
   meson setup build -Dvulkan-drivers=virtio -Dplatforms=x11,wayland -Dgallium-drivers= -Dglx=disabled
   meson compile -C build
   ```
   The missing Python module (`packaging`) was added to the venv (`pip install packaging`, no sudo needed).
   **273/273 targets built successfully.** Output: `build/src/virtio/vulkan/libvulkan_virtio.so` (the client-side Venus/vtest Vulkan ICD).

3. **Important finding:** the source code has a separate file called `vn_renderer_vtest.c` — besides a real virtio-gpu kernel device, Mesa's Venus driver **can also connect as a client to the vtest Unix socket**. To trigger it:
   - `VN_DEBUG=vtest` (force the vtest fallback)
   - `VTEST_SOCKET_NAME=/path/to.sock` (which socket to connect to)
   - `VK_ICD_FILENAMES=.../virtio_devenv_icd.x86_64.json` (a "devenv" ICD file pointing at the .so in the build directory without installing anything — meson generates it automatically, no `meson install` needed)

## End-to-end attempt result: PARTIAL SUCCESS

- `virgl_test_server --venus` (the host, built in the previous round) was started against the local AMD GPU.
- Tried with `vkcube` → `VK_KHR_surface` not found error (a WSI/window problem, a separate issue).
- Tried with `vulkaninfo --summary` (needs no window, only instance/device enumeration):
  - On the first attempt the ICD path was wrong (`/usr/local/lib/...` — the file did not exist because nothing was installed). **Fix:** used the `virtio_devenv_icd.x86_64.json` that meson generates automatically, which points at the real .so path in the build directory.
  - Tried again with the correct ICD: **the client really connected to the server** (the earlier "ICD not found" error was gone), but the server side reported **`VTEST_CLIENT_ERROR_CONTEXT_FAILED`** and `vulkaninfo` dumped core.

## What it means

This is an important threshold: **it is no longer a path/config problem but a real error at
the Venus protocol level.** The ICD loaded and connected to the socket, but creating the
Vulkan context on the host side fails. Possible causes (not verified, hypotheses for the
next round):
1. Venus host support on AMD/RADV may not be as mature as on Intel (ANV)/NVIDIA — upstream virglrenderer/Venus development is tested largely on Intel hardware.
2. A missing flag/context type may not be passed to the `virgl_test_server` invocation.
3. A protocol version mismatch (the venus-protocol version pulled by Mesa may differ from the one pulled by virglrenderer — the two were cloned separately and may not be at the same commit).

## Concrete Tasks for the Next Round
1. Run `virgl_test_server` with a flag such as `-v` (verbose), or with more detailed debug environment variables such as `VN_DEBUG=vtest,init,result`, to find the exact cause of the context error.
2. Check whether the `venus-protocol` subproject commits pulled by Mesa and virglrenderer are the same (`subprojects/venus-protocol-1.1` exists in both repos; a version difference could break context negotiation).
3. Instead of AMD/RADV, if the system has another GPU/driver path (it does not, the only GPU is AMD) — so this debugging should be repeated on AWS/a real NVIDIA GPU to tell whether the problem is specific to RADV.
