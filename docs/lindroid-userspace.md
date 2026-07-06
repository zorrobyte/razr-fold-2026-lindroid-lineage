# Lindroid userspace on LineageOS 23.2 — forward-port status

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

`vendor/lindroid` is the Android-side app + HAL glue (Composer, EventHub-visible input glue,
etc.), `external/lxc` is the container runtime, `external/libhybris` is the Android-GPU-blob
compat shim, `vendor/extra` carries `build/` hooks and product makefiles.

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

| Interface | lindroid-22.1 pin | Android 16 has up to |
|---|---|---|
| `android.hardware.graphics.common` | `V5` | `V7` |
| `android.hardware.graphics.composer3` | `V1` | `V4` |

The `graphics.common` bump is fixed: `vendor/lindroid/interfaces/composer/Android.bp`'s
`imports` now pulls `-V7` instead of `-V5`.
Patch: [`patches/lindroid-aidl-graphics-common-v7.patch`](../patches/lindroid-aidl-graphics-common-v7.patch).

The `composer3` bump (libhybris's `compat/hwc2/Android.mk` still links against
`android.hardware.graphics.composer3-V1-ndk`) is **not yet fixed** — open item.

HIDL composer (`@2.1`–`@2.4`) still ships in Android 16 alongside the AIDL interfaces, so no
rewrite of the HIDL-facing code was needed — only the AIDL version pins.

## Status

An agent-assisted pass got the tree to **analysis-clean** (Soong accepts the `Android.mk`
files, the `graphics.common` bump resolves) and the affected modules **compiling**. Not yet
done:

- Bump `composer3-V1` → a current version in `libhybris/compat/hwc2/Android.mk`.
- Wire `vendor/lindroid/lindroid.mk` into `device/motorola/blanc/device.mk` (currently not
  referenced there at all — `grep -rn lindroid device/motorola/blanc/*.mk` finds nothing).
- A full `blanc` build with Lindroid included, and a boot/smoke test.

This track is currently gated behind the kernel bring-up
([lindroid-kernel.md](lindroid-kernel.md)) anyway — the userspace pieces (LXC, libhybris,
EVDI-facing glue) need `CONFIG_DRM_LINDROID_EVDI`, SYSVIPC, and the namespace configs from that
kernel to do anything, so there's little value in finishing the userspace wiring before the
kernel boots.
