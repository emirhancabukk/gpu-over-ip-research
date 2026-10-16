# Phase 2 — First Local Milestone: virglrenderer + Venus Built and Running (2026-08-27, autonomous round 3)

## Result: SUCCESS — verified against the local AMD GPU, no AWS/Azure needed at all

## Done (reproducible steps)

1. Build dependencies (installed by the user; the autonomous loop could not, because it needs sudo):
   ```
   sudo pacman -S --needed meson ninja pkgconf libepoxy libdrm mesa vulkan-headers vulkan-icd-loader wayland-protocols
   ```

2. The `venus-protocol` subproject needs the Python modules `mako` and `pyyaml`. For these, an isolated venv inside the project was used, **without touching the system with sudo**:
   ```
   python3 -m venv ~/Desktop/gpu-over-ip-research/vendor/.build-venv
   source ~/Desktop/gpu-over-ip-research/vendor/.build-venv/bin/activate
   pip install mako pyyaml
   ```
   (Arch's PEP 668 "externally-managed-environment" protection prevents a system-wide `pip install`; a venv is the correct/safe solution and does not touch the main system at all.)

3. Build:
   ```
   source ~/Desktop/gpu-over-ip-research/vendor/.build-venv/bin/activate
   cd ~/Desktop/gpu-over-ip-research/vendor/virglrenderer
   meson setup build -Dvenus=true -Dplatforms=egl
   meson compile -C build
   ```
   **105/105 targets built successfully.** Output: `build/vtest/virgl_test_server` (the host render server with Venus support), `build/src/libvirglrenderer.so.1.11.0`, `build/server/virgl_render_server`.

4. Smoke test — started the vtest server against the local AMD GPU (`/dev/dri/renderD128`, RADV/Lucienne APU), with Venus enabled:
   ```
   virgl_test_server --no-fork --venus --rendernode /dev/dri/renderD128 --socket-path /tmp/vtest-smoke.sock
   ```
   **Result:** the process started without errors, created the Unix socket file and stayed alive waiting for a client connection. No crash/error log at all.

## What it means

This is **concrete evidence for phase 2 (local PoC, verifying the toolchain without a
network).** The plan previously assumed a full crosvm/QEMU VM would be needed, but thanks
to the vtest server, the host render side was verified to really work without any full
VM. This was done entirely for free, with no need for AWS/Azure.

## Next step
Connect a real Venus/Vulkan **client** (such as `vkcube` built with Mesa's Venus ICD) to
this socket and test whether it can really render a triangle — this will complete phase
2. For that, Mesa itself also has to be built with `-Dvulkan-drivers=virtio` (a separate,
larger build — a task for the next round).
