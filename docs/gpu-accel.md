# GPU acceleration for Lindroid — where this stands

This project's endgoal is a **GPU-accelerated** Lindroid desktop, not the software-rendered
VNC fallback. The deepest investigation of this problem so far was done on the earlier
**stock**-Android effort, not on Lineage yet (Lineage hasn't reached this point — it's still
blocked on the kernel, see [lindroid-kernel.md](lindroid-kernel.md)). This page summarizes
that prior art accurately, without duplicating it; full detail is in
[razr-fold-2026-lindroid](https://github.com/zorrobyte/razr-fold-2026-lindroid)'s
`docs/native-display.md` and `docs/native-turnip.md`.

There are two candidate architectures. Neither is a clean, finished "dead end" — both have real
proven pieces and real open blockers.

## Path A: libhybris (render through the Android/Adreno GPU blob)

`create-disp` (the container-side EVDI broker) allocates real Android gralloc buffers, kwin
renders into them via libhybris's Android-EGL bridge, and the result is HWC2-presented to
Android's compositor. Status per `native-display.md`, **layer by layer**:

1. Kernel EVDI `open()` → `EINVAL` — **fixed** (`FOP_UNSIGNED_OFFSET` missing from `evdi_fops`).
2. `create-disp` finds and opens the EVDI card — **OK**.
3. libhybris missing bionic fortify/stack-protector hooks (`__stack_chk_fail` unhooked →
   SIGSEGV) — **fixed** (rebuilt `libhybris-common` with the hooks added).
4. **Open crux:** after a *retracted* wrong diagnosis (a "wayland-server" theory that turned
   out to describe dead code), the corrected picture is that kwin's DRM backend calls
   `eglCreatePlatformWindowSurfaceEXT(eglDisplay, config, gbm_surface, …)` against a **GBM
   device**, and it is unresolved whether libhybris routes that into
   `lindroid_drmws_CreateWindow` (a vestigial wayland code path that would crash) or whether
   `libgbm-hybris` correctly unwraps the `gbm_surface` to the real Android native window
   underneath (which should work). This requires a real kwin DRM-backend backtrace on-device
   with KCrash disabled to resolve, and that was the immediate next step when the stock effort's
   devices were torn down.
5. A verified **fallback**: pure Mesa software GL (`llvmpipe`) renders fine on the EVDI card —
   proves the EVDI/HWC2 present path itself is sound; only the *hardware-accelerated* render
   path is blocked.

## Path B: Turnip/freedreno directly over KGSL (no libhybris in the render path)

Render with upstream Mesa (Turnip Vulkan / freedreno GL via zink) talking directly to the
stock `/dev/kgsl-3d0` device, bypassing libhybris and the Android EGL blob entirely. This
device's GPU is Adreno 829 (an a8xx part), which Mesa main already has GPU-ID data for. Per
`native-turnip.md`, this is **considerably further along than "Turnip is a dead end" implies**:

| Piece | State |
|---|---|
| Turnip enumerates the real GPU over stock KGSL, in-container | ✅ proven |
| Turnip renders + exports a correct dma-buf | ✅ proven |
| OpenGL ES 3.2 via zink→turnip on the EVDI gbm device (the kwin path) | ✅ proven (needs a 1-line zink patch for pdev selection) |
| A libhybris-free present bridge (native `AHardwareBuffer` → `ASurfaceControl` → SurfaceFlinger) | ✅ proven at the compositor level (real SF layer, frames flowing) |
| A full **visible** GPU desktop (Plasma/sway) | ❌ blocked |

The blockers for Path B are two separate, real problems, **not** insufficient Mesa extension
coverage on the primitives that were tested:

1. **Motorola Ready For owns the external/cover display.** Frames land on a real SurfaceFlinger
   layer, but the physical panel is HWC-marked a "Follower"/inactive display because Ready For's
   `SecondaryDisplayLauncher` composites its own desktop there (`ro.product.motodesktop=1`);
   force-stopping it just respawns. Needs either drawing directly via `libgui`/`SurfaceControl`
   at top z-order, or reconfiguring so the external display isn't Ready-For-managed.
2. **Adreno-8xx support in Mesa main is very new and rough at the full-compositor level.**
   Offscreen/surfaceless GPU work (enumerate, render, export, GL-via-zink on the EVDI gbm
   device) all pass. But real compositors fail: kwin's virtual backend hits
   `GL_FRAMEBUFFER_INCOMPLETE_ATTACHMENT`, sway/wlroots crashes, and turnip logs a
   `vkGetCalibratedTimestampsEXT VK_ERROR_OUT_OF_HOST_MEMORY` spam (likely an a8xx turnip bug).

## Why this project is pursuing the libhybris path anyway

The userspace forward-port work done so far (`vendor/lindroid`, `libhybris`, `external/lxc` —
see [lindroid-userspace.md](lindroid-userspace.md)) follows the **libhybris** architecture, the
same one the stock effort used, rather than the Turnip/KGSL path. The reasoning: libhybris is
the architecture Linux-on-droid's upstream Lindroid is built around, so it's the path with the
most existing integration (composer glue, HWC2 present, LXC config) to forward-port instead of
build from scratch. The Turnip/KGSL path in `native-turnip.md` is a from-scratch alternative
render pipeline the stock effort prototyped in parallel; it is *not* wired into `vendor/lindroid`
today and would need its own integration work (replacing `create-disp`'s buffer allocation and
kwin's EGL platform) regardless of which is closer to working.

## What Lineage changes here (and what it doesn't)

Lineage's advantage over the stock effort is **debuggability** — root, clean coredumps, a
patchable framework — which shortens the loop for resolving either crux above. It does not
change the crux code itself: whichever of the two paths gets a GPU-accelerated desktop working
will be decided by the same libhybris/kwin/GBM or Mesa-a8xx-compositor code documented in the
stock repo. This Lineage effort hasn't reached the point of re-attempting either path yet — it
is currently blocked earlier, at the kernel level (see [lindroid-kernel.md](lindroid-kernel.md)).
