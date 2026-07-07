# Lindroid userspace on LineageOS 23.2 — forward-port, build, boot

**Status: forward-ported, built, flashed, booted.** The Lindroid Android-side userspace
(`vendor/lindroid`, `external/lxc`, the needed slice of `external/libhybris`) now builds against
`lineage-23.2` and is present on a running `blanc` device. Runtime behavior of the container and
first display output are covered separately in [display-bringup.md](display-bringup.md); this
page covers the forward-port and the Android-build-side wiring that got it there.

Bringing the [Linux-on-droid](https://github.com/Linux-on-droid) Lindroid userspace (LXC guest
+ libhybris compat layer + Android-side glue) into the Android-16 tree. **No Lindroid branch
targets Android 16 upstream** — the newest published branch is `lindroid-22.1` /
`lindroid-21` (Android 12/13-era), so this is a forward-port, not a drop-in.

## Repos added

Via `.repo/local_manifests/lindroid.xml` (copied to
[`.repo-local-manifest/lindroid.xml`](../.repo-local-manifest/lindroid.xml) in this repo):

```xml
<manifest>
  <remote name="lindroid" fetch="https://github.com/Linux-on-droid/" />

  <project name="vendor_lindroid" path="vendor/lindroid" remote="lindroid" revision="lindroid-22.1" clone-depth="1" />
  <project name="external_lxc" path="external/lxc" remote="lindroid" revision="lindroid-21" clone-depth="1" />
  <project name="libhybris" path="external/libhybris" remote="lindroid" revision="lindroid-21" clone-depth="1" />
  <project name="vendor_extra" path="vendor/extra" remote="lindroid" revision="lindroid-21" clone-depth="1" />
</manifest>
```

`vendor/lindroid` is the Android-side app + HAL glue (`perspectived`, `LindroidUI`, the
composer/perspective HALs), `external/lxc` is the container runtime, `external/libhybris` is the
Android-GPU-blob compat shim, `vendor/extra` carries `build/` hooks and product makefiles.

## Forward-port result: what builds against A16

| Component | Result |
|---|---|
| `external/lxc` — all 23 `lxc-*` command-line tools (`lxc-start`, `lxc-attach`, `lxc-create`, `lxc-ls`, `lxc-info`, …) | ✅ build clean |
| `vendor/lindroid` — `perspectived`, `LindroidUI`, composer HAL, perspective HAL | ✅ build clean |
| `external/libhybris` — every module referenced by `vendor/lindroid/lindroid.mk`'s `PRODUCT_PACKAGES` | ✅ build clean |
| `external/libhybris` — 11 other modules (the `sf`/`media`/`camera`/`input` compat shims, the `_org` legacy-ABI variants, a gralloc test binary) | 🔴 fail to build |

The 11 failing libhybris modules are **not a blocker**: none of them are referenced by
`lindroid.mk`'s `PRODUCT_PACKAGES`, so Soong never needs to build them for a `blanc` system
image — they're unreferenced legacy code left over from older, non-`vendor/lindroid`-based
Lindroid integrations (direct `SurfaceFlinger`/`CameraService` compat, superseded by the AIDL
HAL glue `vendor/lindroid` now provides). They were left broken rather than fixed or deleted, so
future upstream merges stay simple.

## Fix: `Android.mk` denylist

Newer Soong (Android 16's build system) refuses to process `Android.mk` files under
`external/` by default (a hardening measure against legacy `Make`-based builds). Six of
libhybris's compat modules are still `Android.mk`-based:

```
external/libhybris/compat/ui/Android.mk
external/libhybris/compat/input/Android.mk
external/libhybris/compat/camera/Android.mk
external/libhybris/compat/surface_flinger/Android.mk
external/libhybris/compat/media/Android.mk
external/libhybris/compat/hwc2/Android.mk
```

The build already has the intended hook for this: `vendor/extra/build/androidmk/allowlist.txt`.
Populating it with those six paths is the fix — no Soong/Kati changes needed.

Patch (new file): [`patches/vendor-extra-androidmk-allowlist.patch`](../patches/vendor-extra-androidmk-allowlist.patch).

## Fix: AIDL version pins vs. Android 16

Android 16 ships newer major versions of the graphics AIDL interfaces than `lindroid-22.1`'s
`vendor/lindroid` was written against:

| Interface | lindroid-22.1 pin | Android 16 has up to | Status |
|---|---|---|---|
| `android.hardware.graphics.common` | `V5` | `V7` | ✅ fixed — bumped to `V7` |
| `android.hardware.graphics.composer3` | `V1` | `V4` | ✅ fixed — bumped to `V4` |

`vendor/lindroid/interfaces/composer/Android.bp`'s `imports` now pulls `-V7` instead of `-V5`.
Patch: [`patches/lindroid-aidl-graphics-common-v7.patch`](../patches/lindroid-aidl-graphics-common-v7.patch).

`external/libhybris/compat/hwc2/Android.mk` now links `android.hardware.graphics.composer3-V4-ndk`
instead of `-V1-ndk` (this was the "open item" from the previous pass through this doc — it's
closed now).
Patch: [`patches/lindroid-composer3-v4.patch`](../patches/lindroid-composer3-v4.patch).

HIDL composer (`@2.1`–`@2.4`) still ships in Android 16 alongside the AIDL interfaces, so no
rewrite of the HIDL-facing code was needed — only the AIDL version pins.

## Fix: `HdrCapabilities` vector cast in `HWC2.cpp`

`external/libhybris/compat/hwc2/HWC2.cpp`'s HDR-capability query builds its result from the AIDL
`Hdr` enum's underlying integer values without an explicit cast — a narrowing conversion that
`lindroid-22.1`'s toolchain tolerated silently but Android 16's stricter Clang rejects as a hard
error. Fixed with an explicit `static_cast<int32_t>` per element when constructing the returned
vector, rather than relaxing the warning level for the whole file.

## Fix: `android-config.h` stub

Parts of `external/libhybris` (autotools-derived from upstream libhybris, before the
`Android.bp`/Soong port) expect a generated `android-config.h` defining the
`ANDROID_VERSION_MAJOR`/`ANDROID_VERSION_MINOR`/API-level macros that a `./configure` step would
normally produce. Soong has no such step. Fixed with a small checked-in stub header (values
pinned for Android 16 / API 36) so the sources that `#include` it compile under Soong.

## Fix: `lindroid.mk` wiring into `device.mk`

Even with the forward-port compiling cleanly, `vendor/lindroid/lindroid.mk` was never actually
included by the `blanc` device tree — `grep -rn lindroid device/motorola/blanc/*.mk` found
nothing. Wired in via `$(call inherit-product, vendor/lindroid/lindroid.mk)` in
`device/motorola/blanc/device.mk`.

That surfaced a second, non-obvious build break: `soong_filesystem_creator` rejected
`.../configs/disabled.idc` as **"outside source root."** Root cause: `lindroid.mk` computed its
own directory with `LOCAL_PATH := $(call my-dir)`, which relies on
`$(lastword $(MAKEFILE_LIST))` to find the currently-executing makefile's path — a trick that
only works when Make is walking the file directly. Reached via `inherit-product` from a product
makefile instead, `my-dir` resolves to an **empty string**, so every `$(LOCAL_PATH)/configs/...`
reference in the file quietly became a bare, source-root-relative (i.e. wrong) path. Fix:
hardcode it, since the path is fixed by the repo manifest anyway.

```diff
-LOCAL_PATH := $(call my-dir)
+LOCAL_PATH := vendor/lindroid
```

Patch: [`patches/lindroid-mk-local-path.patch`](../patches/lindroid-mk-local-path.patch).

## Fix: `perspectived` sepolicy — neverallow violation

With `lindroid.mk` wired in, `perspectived`'s domain shipped `allow perspectived init:dir
w_dir_perms;`. That trips the platform-wide neverallow:

```
neverallow domain domain:dir { write add_name remove_name };
```

This is a **compile-time** sepolicy check — unlike an ordinary denial, being in permissive mode
does not save you, since the policy fails to link before anything runs, and the whole system
build breaks. `perspectived` only needs to *read* `init`'s directory (e.g. to locate init's
well-known sockets); it never writes into it. Fix: downgrade to `r_dir_perms`.

```diff
-allow perspectived init:dir w_dir_perms;
+allow perspectived init:dir r_dir_perms;
```

Patch: [`patches/perspectived-sepolicy-r-dir-fix.patch`](../patches/perspectived-sepolicy-r-dir-fix.patch).

## Built, flashed, booted

With the fixes above, a full `blanc` build (`system`, `system_ext`, `product`) with Lindroid
included was built, flashed per the same milestone-1 strategy as the rest of this port (see
[lineageos-port.md](lineageos-port.md)), and **booted**:

- `org.lindroid.ui` (`LindroidUI`) installed and launchable.
- All Lindroid components staged on-device: `perspectived`, the `lxc_*` binaries,
  `LindroidUI.apk`, and the hacked `libc.so` at `/system_ext/usr/share/lindroid/` (libhybris's
  bionic-compat `libc` used inside the container's namespace, distinct from Android's own
  `libc.so`).

**Known caveat:** `perspectived` does not auto-start at boot. Its binary landed labeled
`unlabeled` rather than a proper `system_ext`/vendor sepolicy label — a `file_contexts`
packaging gap between where the binary is installed (`system_ext`) and where `vendor/lindroid`'s
own `file_contexts` entries assume it lives (a `vendor`-style layout). Started manually for now
(`su -c /system_ext/bin/perspectived &` equivalent); proper fix is adding the `system_ext`-side
`file_contexts` entry, tracked as an open item alongside the SELinux-enforcing work below.

For what happens once `perspectived` is running and a container is started — the storage
workaround, the first real display output, and the current GPU-desktop blocker — see
[display-bringup.md](display-bringup.md).

## Status / open items

Forward-port, build wiring, and first boot are done. Remaining, in rough priority order:

- **`perspectived` file_contexts / auto-start** — see caveat above.
- **Container storage: raw ext4 loop image instead of hand-edited `dir:` rootfs** — see
  [display-bringup.md](display-bringup.md)'s storage section; this is what currently makes
  every container need manual config editing before it will start.
- **SELinux enforcing for the new domains** — bring-up is running with SELinux globally
  permissive (see [display-bringup.md](display-bringup.md)); `perspectived`/`lxc_*`/composer
  domains need real policy before flipping back to enforcing.
- **GPU-accelerated desktop** — blocked on an EVDI kernel bug independent of anything in this
  doc; see [display-bringup.md](display-bringup.md) and [gpu-accel.md](gpu-accel.md).


## Note: perspectived is a coredomain mislabeled in vendor sepolicy

`perspectived` (the Lindroid container/display supervisor) is declared as a
`coredomain` but its type/`file_contexts` live in **vendor** sepolicy
(`vendor/lindroid/sepolicy`). A coredomain must be defined in
`system`/`system_ext` (plat/plat_pub) sepolicy, not vendor. Because of the
split, init cannot resolve the process context at autostart time and the
service fails to launch with:

```
init: Could not get process context for service perspectived ... (No such file or directory)
```

**Fix direction (not yet applied):** move the `perspectived` type definition,
`.te`, and `file_contexts` entry out of `vendor/lindroid/sepolicy` and into
`system_ext` (or plat) sepolicy so the coredomain label is visible to init's
core policy. Until then perspectived is started manually / via a shim rather
than init-autostarted. Tracked here so it isn't lost; the `r_dir_perms`
neverallow fix (see `patches/perspectived-sepolicy-r-dir-fix.patch`) is
independent of this labeling problem.
