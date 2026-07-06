# razr-fold-2026-lindroid-lineage

LineageOS 23.2 (Android 16) on the Motorola Razr Fold 2026 (`blanc`, SM8845 / `canoe`) — and
the effort to run **Lindroid** (Linux-in-LXC with hardware-accelerated display) on top of it.

This is a documentation + patch companion repo. It does not contain a buildable tree; it
records what was done to the trees in the companion repos below, with real diffs extracted
from the working checkouts.

## Device

| | |
|---|---|
| Product / codename | `blanc_gu` / **blanc** |
| SKU | XT2651-2 |
| SoC | Qualcomm **SM8845** (`canoe`) |
| GPU | Adreno 829 (a8xx family) |
| Form factor | Foldable, inner + cover panel (cover panel 165 Hz capable) |
| Stock build | W3WBS36.36-48-5-1 |
| Stock kernel | GKI `6.12.38-android16-5` |
| Target | LineageOS **23.2** (Android 16) — newest branch at the time of this work; 24.0 = Android 17 |
| Status | **First SM8845/canoe device on LineageOS** |

## Status ladder

| Layer | State |
|---|---|
| LineageOS boots, SELinux enforcing | ✅ done ("milestone 1" strategy, see [docs/lineageos-port.md](docs/lineageos-port.md)) |
| Display / touch / UI | ✅ working |
| Wi-Fi, RIL, NFC, Bluetooth | ✅ working |
| Sensors incl. Moto hinge posture | ✅ working |
| GPS provider | ✅ working |
| Audio HALs | ✅ working |
| Fingerprint (goodix HAL) | ✅ running |
| Rear + inner-selfie cameras | ✅ working |
| Cover-panel 165 Hz refresh | ✅ fixed, see [docs/fixes.md](docs/fixes.md) |
| Folded selfie camera | ✅ fixed, see [docs/fixes.md](docs/fixes.md) |
| Lindroid input isolation (EventHub `.idc`) | ✅ ported, see [docs/fixes.md](docs/fixes.md) |
| Lindroid userspace (vendor/lindroid, libhybris, lxc) on A16 | ✅ forward-ported, built, flashed — see [docs/lindroid-userspace.md](docs/lindroid-userspace.md) |
| Lindroid kernel (SYSVIPC/namespaces/EVDI) | ✅ **SOLVED**, boot-verified — see [docs/lindroid-kernel.md](docs/lindroid-kernel.md) |
| Lindroid container boots, first display output on physical screen | ✅ achieved (Debian/KDE container, EVDI → composer → LindroidUI) — see [docs/display-bringup.md](docs/display-bringup.md) |
| GPU-accelerated Lindroid desktop | 🔴 blocked on an EVDI `open()` EINVAL in the currently-flashed kernel release (fix identified, kernel rebuild in progress) — see [docs/display-bringup.md](docs/display-bringup.md) and [docs/gpu-accel.md](docs/gpu-accel.md) |

## Companion repos

- **[android_device_motorola_blanc](https://github.com/zorrobyte/android_device_motorola_blanc)**
  (branch `lineage-23.2`) — the LineageOS device tree for `blanc`.
- **[razr-fold-2026-kernel-build](https://github.com/zorrobyte/razr-fold-2026-kernel-build)** —
  from-source kernel build harness, tag-pinned to Moto's OSS release `MMI-W3WB36.36-48-5`.
- **[razr-fold-2026-lindroid](https://github.com/zorrobyte/razr-fold-2026-lindroid)** — the
  earlier **stock-Android** Lindroid effort (Magisk-based). Its `docs/native-display.md` and
  `docs/native-turnip.md` are the deepest existing writeup of the GPU-acceleration problem and
  are referenced from this repo rather than duplicated.

## How this differs from the stock effort

The stock repo above got Lindroid running under stock Android via Magisk modules, smali-patched
`services.jar`, and a prebuilt/reused kernel — because stock Android gives no other angle in.
This repo is the LineageOS-based successor, which changes the leverage available:

- **Source-level fixes instead of binary patching.** The stock effort's `services.jar` hacks
  (`canJoinSharedUserId` smali patch, `WiredAccessoryManager` null-guard smali patch — see its
  `docs/services-jar.md`) exist only because stock Android can't be recompiled. On Lineage the
  same class of problem (here, isolating Lindroid's virtual input devices from Android apps) is
  fixed directly in `frameworks/native` and rebuilt — no baksmali/smali, no odex/vdex whiteout
  dance, no Magisk-safe-mode bootloop risk. See the EventHub `.idc` fix in
  [docs/fixes.md](docs/fixes.md).
- **A rebuildable kernel instead of a frozen prebuilt.** The stock effort reused a prebuilt
  Lindroid kernel as-is. Here the kernel is built from the same tree as the (from-source, tag-
  pinned) stock-equivalent kernel, so the Lindroid deltas (namespaces, EVDI, SYSVIPC) could be
  iterated and diffed against a known-good baseline — which is exactly how an early boot panic
  in this effort was root-caused and then fixed for good (see
  [docs/lindroid-kernel.md](docs/lindroid-kernel.md)).
- **Debuggability.** Root, clean coredumps, and a patchable `WindowManager`/`CameraServiceProxy`
  are available by default. The stock effort's `native-display.md` diagnosis (libhybris TLS
  patcher vs. Android-16 bionic, then the kwin GBM/EGL surface-creation crux) was hard-won under
  Magisk; on Lineage the same crux is still open, but the debugging loop is far shorter.
- **The crux code itself hasn't changed.** Whether the GPU-accelerated desktop finally renders
  will most likely be decided by the same libhybris/kwin/GBM code paths documented in the stock
  repo, not by anything Lineage-specific. See [docs/gpu-accel.md](docs/gpu-accel.md).

## Layout

```
docs/
  lineageos-port.md      milestone-1 strategy, flash recipe, hardware survey
  fixes.md               refresh rate / folded camera / inputflinger fixes, with patches
  lindroid-kernel.md      SOLVED: the boot-verified release, the two ABI tricks, reproduce-from-source
  lindroid-userspace.md   vendor/lindroid + libhybris + lxc forward-port to A16, build + boot
  display-bringup.md      first display output, container storage fix, the EVDI EINVAL desktop crux
  gpu-accel.md            summary of the GPU-acceleration investigation (Turnip vs libhybris)
patches/
  camera-folded-selfie.patch          frameworks/base — DEVICE_STATE_FRONT_COVERED gating
  inputflinger-idc-disable.patch      frameworks/native — EventHub .idc device.disabled
  lindroid-aidl-graphics-common-v7.patch   vendor/lindroid — AIDL version bump for A16
  lindroid-composer3-v4.patch         external/libhybris — composer3 AIDL version bump for A16
  lindroid-mk-local-path.patch        vendor/lindroid — LOCAL_PATH fix for product-inherited lindroid.mk
  perspectived-sepolicy-r-dir-fix.patch    vendor/lindroid sepolicy — neverallow-safe perspectived rule
  vendor-extra-androidmk-allowlist.patch   vendor/extra — Android.mk denylist allowlist hook
  kernel/version.c.patch              kernel: force-load stock modules past module_layout CRC shift
  kernel/sched.h-kabi.patch           kernel: relocate sysvsem/sysvshm into ANDROID_KABI_RESERVE
  kernel/ipc-exports.patch            kernel: EXPORT_SYMBOL(put_ipc_ns)/EXPORT_SYMBOL(init_ipc_ns)
  kernel/lindroid_gki.fragment        kernel: base-kernel defconfig fragment (namespaces + EVDI)
.repo-local-manifest/
  lindroid.xml            the repo local_manifest that pulls in vendor/lindroid + friends
```
