# Lindroid display bring-up — first pixels, and the desktop blocker

This page picks up where [lindroid-userspace.md](lindroid-userspace.md) leaves off: the Android
build boots with Lindroid staged, `perspectived` running (started manually — see that doc's
caveat), and a container that has been hand-configured to actually start (see "Container storage"
below). It covers what happens at runtime: the first-ever display output on this device, the two
blockers found along the way, and current caveats.

## Milestone: first-ever display output

The container's own framebuffer console rendered through the full Lindroid pipeline and appeared
**on the phone's physical screen**:

```
container framebuffer (Debian console)
  -> EVDI virtual display (/dev/dri/cardN, kernel)
  -> vendor.lindroid.composer (Android HAL side)
  -> LindroidUI (org.lindroid.ui)
  -> phone screen
```

This is past the historical black-screen/crash wall this class of bring-up usually hits first.
It is **not** yet the GPU-accelerated desktop this project is aiming for — see "The desktop
crux" below — but it proves every piece of the pipeline *except* hardware-accelerated rendering
is wired correctly end to end: EVDI is producing frames, the composer HAL is picking them up,
LindroidUI is presenting them, and Android's compositor is putting them on the panel.

The audio bridge (`/lindroid/audio_socket`) also connected successfully in the same session —
container-side audio has a working path out to the Android audio HAL.

### Container environment

The created container is **Debian 13 (trixie), arm64**, with a full KDE Plasma 6 stack
(`kwin_wayland`, `sddm`) plus the `libhybris`/`gbm-hybris`/`drihybris` GPU compat stack already
installed inside it — this is the desktop the project is trying to get GPU-accelerated (see
below).

A root shell was obtained via `lxc-attach` into the running container. All required namespaces
(`NEWNS`/`PID`/`UTS`/`IPC`/`CGROUP`) work correctly, and both `/dev/kgsl-3d0` (the Adreno GPU
node) and `/dev/dri` are passed through into the container as expected.

## Blocker 1 (worked around): container storage vs. f2fs casefold

The Lindroid app generates container configs with:

```
lxc.rootfs.path = overlayfs:/data/lindroid/...
```

But `/data` on this device is `f2fs` with casefold enabled, and the kernel's `overlayfs` refuses
to mount on top of a case-insensitive-capable filesystem:

```
EINVAL: case-insensitive capable filesystem not supported
```

**Workaround (current):** hand-edit the generated container config, replacing the
`overlayfs:` rootfs line with a plain `dir:` rootfs pointing at an already-extracted directory
tree. This works, but it means **every container currently has to be created and then manually
patched before it will start** — there's no story yet for doing this through the normal
LindroidUI container-creation flow.

**Proper fix (TODO, not yet built):** a raw ext4 loop image for container storage instead of an
overlay on the live `/data` filesystem — the same approach Waydroid uses for its own container
storage, and one that sidesteps the f2fs-casefold interaction entirely regardless of what `/data`
is formatted as.

## Blocker 2 (root-caused, fix pending reflash): EVDI `open()` → `EINVAL`

This is the actual crux blocking a GPU-accelerated desktop right now.

### Symptom

Both `create-disp` (the container-side EVDI broker) and `kwin_wayland` fail identically:

```
open("/dev/dri/cardN", O_RDWR) = -1 EINVAL
```

on **every** `evdi-lindroid` card. The real Adreno GPU node (`/dev/dri/card0`) opens fine — this
is specific to the EVDI virtual-display cards. Confirmed on the **host** side too (not a
container/namespace/permission artifact, and not a composer- or libhybris-level bug): a plain
`open()` from outside the container against an `evdi-lindroid` card node fails the same way.

A visible side effect of the failure mode: `create-disp` loops, logging `"evdi-lindroid still
not available after add attempt"` and **leaking a new EVDI card on every retry** — `card1`,
`card2`, ... up past `card15` observed in one run — with every connector reporting
`disconnected`.

### Root cause

Linux 6.12's `drm_open_helper()` returns `-EINVAL` unless the driver's registered
`file_operations` sets `.fop_flags = FOP_UNSIGNED_OFFSET`. This is a real, if obscure, kernel
requirement introduced for 6.12 — not an EVDI-specific bug, just a compat flag the EVDI driver
needs to carry for this kernel version.

**This is not a new diagnosis** — it's already documented as fix #3 in
[lindroid-kernel.md](lindroid-kernel.md)'s writeup of the "solved" kernel release, and it's the
same crux [gpu-accel.md](gpu-accel.md) lists as "fixed" under Path A step 1. What this session's
testing found is a **discrepancy between the vendored driver source and the currently-flashed
release**:

- The driver source in the kernel-build tree
  (`kernel-build/lindroid/evdi/evdi_lindroid_drv.c`, around lines 39–41) **already has**
  `.fop_flags = FOP_UNSIGNED_OFFSET` set.
- The **currently-flashed** release
  (tag `lindroid-w3wb36.36-48-5`, commit `7fa8018`, see
  [lindroid-kernel.md](lindroid-kernel.md)) was built from a tree state that **predates** that
  fix landing in source. In other words: the fix exists upstream in the kernel-build repo, but
  the specific boot image currently on the device was built before it was added, so the bug it
  fixes is still reproducing on real hardware.

> **Correction to lindroid-kernel.md:** that page's compat-fix list for the EVDI driver
> (`DRM_UNLOCKED` shim, `void`-returning `platform_driver::remove`, `FOP_UNSIGNED_OFFSET`)
> describes the driver source accurately, but the specific release binary documented there does
> **not** yet contain the `FOP_UNSIGNED_OFFSET` fix — this was verified by direct reproduction of
> the `EINVAL` on real hardware in this session, after the kernel itself (namespaces, EVDI module
> loading, `/dev/dri` node creation) was otherwise confirmed working per that page. The kernel
> bring-up itself is still correctly "solved"; this is a narrower, since-identified gap in the
> specific flashed artifact.

### Fix and status

Rebuild the kernel from the current source (which already carries the fix,
`CONFIG_DRM_LINDROID_EVDI=y`, built-in per the existing recipe) and reflash `boot_a`. **A rebuild
is in progress as of this writing.** No new kernel patch is required — this is a rebuild-and-
reflash of the existing, already-fixed source, not a new source change. Once flashed, re-run the
`create-disp` / `kwin_wayland` path and confirm the EVDI cards open successfully instead of
looping and leaking card nodes.

## Current caveats

- **SELinux is globally permissive** for the duration of this bring-up. This is intentional —
  labeling the new Lindroid domains correctly is being deferred until the functional pieces
  (storage, display, GPU) are proven, rather than debugging policy and functionality at the same
  time. Re-enforcing is tracked in [lindroid-userspace.md](lindroid-userspace.md)'s status list.
- **`perspectived` doesn't auto-start** — see [lindroid-userspace.md](lindroid-userspace.md);
  started manually for every test in this session.
- **Containers must be started through LindroidUI / `perspectived`, not raw `lxc-start`.** Only
  that path sets up the composer socket bridge (`vendor.lindroid.composer` <-> the container's
  compositor); a container started with bare `lxc-start` comes up with no display path at all,
  which looks superficially similar to the EVDI `EINVAL` failure above but is a different,
  trivial-to-avoid problem (make sure you went through the app, not the CLI directly).
- **USB flashing is slow (~1 MB/s)** in the current setup — an active Thunderbolt 4 cable caps
  the phone's USB link at USB-2/Full-Speed, and `fastbootd` doesn't renegotiate faster once
  dropped to that speed. Purely cosmetic/inconvenient, not a functional issue; swapping to a
  USB-C-to-USB-C (non-TB4) cable fixes it.

## Next steps

1. Land the kernel rebuild with the EVDI `FOP_UNSIGNED_OFFSET` fix actually included, reflash,
   and confirm `evdi-lindroid` cards open without `EINVAL`.
2. Re-attempt `kwin_wayland` bring-up on a working EVDI card and see how far the libhybris
   render path (Path A in [gpu-accel.md](gpu-accel.md)) gets now that the kernel-level blocker
   for *this specific device/kernel build* is closed.
3. Build the ext4 loop-image container storage so containers don't need hand-editing.
4. Start re-enforcing SELinux, domain by domain, once the above are stable.
