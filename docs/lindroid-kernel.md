# Lindroid kernel — SOLVED (boot-verified release)

**Status: solved.** A from-source `blanc`/`canoe` kernel boots to home with audio, Bluetooth,
Wi-Fi, NFC, **and** full Lindroid kernel support (LXC container namespaces + the EVDI virtual
display) — while keeping the **stock, proprietary `vendor_dlkm`** (Moto never published those
module sources, so it can't be rebuilt). This page documents the working release, the exact
tricks that get a namespace-enabled kernel past the GKI ABI wall against unmodified stock
modules, and how to reproduce it from source.

> An earlier version of this page described the kernel as an open blocker (an `init`-death panic,
> `exitcode=0x7f00`). That panic was real, but it was caused by flashing the **wrong** kernel
> image — an interim build (from `Ubuntu-24.04:~/kbuild`) that had `CONFIG_SYSVIPC` raw-enabled
> **without** the `task_struct` KABI-preservation patch below, so enabling SysV IPC shifted
> `task_struct`'s layout and the stock `sched_walt` module NULL-derefed against it. The fix isn't
> a workaround for that panic, it's a different, correct build (see "The two ABI tricks" below)
> that was boot-tested and is now published as a release. That panic is resolved.

## The release (flash this)

| | |
|---|---|
| Repo | [razr-fold-2026-kernel-build](https://github.com/zorrobyte/razr-fold-2026-kernel-build) |
| Tag | [`lindroid-w3wb36.36-48-5`](https://github.com/zorrobyte/razr-fold-2026-kernel-build/releases/tag/lindroid-w3wb36.36-48-5) |
| Commit | `7fa8018` ("docs(lindroid): mark boot-verified working") |
| Asset | `boot_lindroid.img` |
| SHA-256 | `145b95476d3a4c6b51b6e6a433ec98457f7eee3a58ae32f97ab33b0494dcfa1b` |

Other assets in the same release: `boot_notrim-recovery.img` (audio-only kernel, no container ABI
changes — flash this to recover if a future change loops), `config-lindroid.txt` (the exact
running `.config`), `Module.symvers` (`vendor_data_pad = 0xf54e5881` — the one CRC that
legitimately differs from stock and is accounted for, see `docs/SOURCES.md`/README in that repo),
and `SHA256SUMS.txt`:

```
145b95476d3a4c6b51b6e6a433ec98457f7eee3a58ae32f97ab33b0494dcfa1b  boot_lindroid.img
d6df5937386a56606786d916ba7d324c82e52f709f7103d2b5467f6f8d95838d  boot_notrim-recovery.img
f4f49401a6837cf7fe07c7683a24720c92d97bca43ea5d797d8957192a8133b9  config-lindroid.txt
db895c9f4353717e4ceb2d305fe6b3faa28ccba9e98071c7c660e5f47db87ac6  Module.symvers
```

### On-device boot verification (already done)

```
sys.boot_completed=1
unshare --user --pid --ipc --uts --fork   -> OK (namespaces work)
/sys/module/evdi  -> version 1.0.0
/dev/dri/card0, /dev/dri/renderD128        -> present (EVDI)
/proc/asound/cards                         -> alorqrdsndcard (audio works)
```
Also verified: Bluetooth, Wi-Fi, NFC all working (same `rfkill`/`btpower`/`cnss2` stock modules as
the plain audio-fix kernel — see the harness README's finding #3 on `--notrim`).

### Flash recipe

Flash **only `boot`**; every other partition (`vendor_boot`, `dtbo`, `vendor_dlkm`, `system_dlkm`,
`odm`, etc.) stays **stock**, from the factory `BLANC_G_W3WBS36.36_48_5` package (LMSA). This is
the same "keep everything but our one modified partition stock" strategy the whole port uses (see
[lineageos-port.md](lineageos-port.md)).

```bash
# download boot_lindroid.img from the release above, verify its sha256 first
sha256sum boot_lindroid.img   # must be 145b95476d3a4c6b51b6e6a433ec98457f7eee3a58ae32f97ab33b0494dcfa1b

adb reboot bootloader
fastboot flash boot_a boot_lindroid.img

# Moto's vbmeta rejects fastboot's --disable-verity flag directly ("Failed to find AVB_MAGIC");
# patch the flags byte instead (see scripts/patch-vbmeta.py in android_device_motorola_blanc),
# or use --disable-verity --disable-verification if your vbmeta accepts it:
fastboot --disable-verity --disable-verification flash vbmeta        <factory>/vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_system <factory>/vbmeta_system.img

# vendor_boot / dtbo / super (vendor_dlkm, system_dlkm) stay STOCK — do not flash the
# from-source vendor_dlkm (it's missing ~126 proprietary modules Moto never published).
# Re-flash the Magisk-patched init_boot afterward to keep root.
fastboot reboot
```

**Recovery if it loops (~30 s):** `fastboot flash boot_a boot_notrim-recovery.img` (audio-only,
no container ABI changes — always boots), or `fastboot flash boot_a <factory>/boot.img` for full
stock, then re-diagnose with `adb shell su -c 'cat /sys/fs/pstore/console-ramoops-0'`.

## Why this is hard: the GKI ABI wall

A stock Android GKI kernel ships with a **frozen, versioned ABI** (KMI) so a from-source kernel
can (in principle) swap in for the shipped one without rebuilding vendor modules. Lindroid needs
`CONFIG_SYSVIPC`, `CONFIG_PID_NS`, `CONFIG_IPC_NS`, `CONFIG_USER_NS`, and friends turned on for LXC
containers — but those options change core kernel structs (that's the whole reason KMI freezes
them in normal GKI). Since Moto never published `vendor_dlkm`'s ~126 proprietary module sources
(display, GPU, Wi-Fi/BT, audio codecs, touch, Moto's own `mmi_*`/`utags`), those modules **cannot
be rebuilt** against the new struct layouts — they have to keep loading, unmodified, against a
kernel whose ABI just changed underneath them. Two separate breakages result, and each has its own
targeted fix.

### Trick 1 — force-load stock modules past the `module_layout` CRC shift

Turning on the namespace configs changes the global module-ABI CRC (`module_layout`:
`0xe976b219` → `0xe351f8ce` on this build). With `CONFIG_MODVERSIONS=y` (kept `=y` deliberately —
see below), the kernel's runtime symbol-version gate normally rejects every stock module:
`"<mod>: disagrees about version of symbol module_layout"` → no storage/display/etc. drivers load
→ first-stage panic.

`MODVERSIONS` is **not** disabled, because that would drop the `modversions` vermagic token and
make every module `ENOEXEC` at the coarser vermagic-string check instead — a worse failure.
Instead, `kernel/module/version.c`'s `check_version()` is patched so its failure path warns and
force-loads rather than rejecting
(`patches/kernel/version.c.patch`, applied by `scripts/build.sh` in the kernel-build repo):

```c
 bad_version:
 	pr_warn("%s: disagrees about version of symbol %s\n", info->name, symname);
-	return 0;
+	return 1; /* lindroid force-load: accept ABI CRC shift from container configs */
 }
```
This is safe specifically because the *only* structs the changed CRCs cover are namespace
internals (`task_struct`, IPC namespace bits) that stock vendor drivers never touch directly —
see Trick 2 for the one struct that stock code *does* read into, and why it needs its own fix
rather than relying on force-loading alone.

### Trick 2 — preserve `task_struct`'s byte layout so `sched_walt` doesn't NULL-deref

Force-loading (Trick 1) makes stock modules *load*, but doesn't make them safe: `CONFIG_SYSVIPC`
normally inserts `struct sysv_sem sysvsem` and `struct sysv_shm sysvshm` inline into `task_struct`,
**before** the `android_vendor_data` reserved-KABI region, which shifts every field after them —
including `android_vendor_data1`, which the stock, unmodified `sched_walt` module reads at a fixed
offset (`task_fits_capacity()`) on every scheduling decision. That shift is exactly what the
earlier, wrongly-flashed interim kernel hit: **NULL-deref panic in `sched_walt` at ~0.6 s**,
manifesting at the `init` level as `Attempted to kill init! exitcode=0x7f00`.

The fix: relocate `sysvsem`/`sysvshm` **out of** their normal inline position and **into** the
`ANDROID_KABI_RESERVE(1..3)` slots — padding the GKI ABI already reserves *after*
`android_vendor_data` specifically so vendor/downstream kernels can add fields without breaking
the frozen KMI. `include/linux/sched.h` (`patches/kernel/sched.h-kabi.patch`, applied via `patch
-p1` by `scripts/build.sh`):

```c
 	struct nameidata		*nameidata;
 
-#ifdef CONFIG_SYSVIPC
-	struct sysv_sem			sysvsem;
-	struct sysv_shm			sysvshm;
-#endif
+	/* lindroid: sysvsem/sysvshm relocated into ANDROID_KABI_RESERVE(1..3) below to keep the
+	 * frozen (SYSVIPC=off) task_struct ABI so stock GKI vendor modules (sched_walt) read
+	 * android_vendor_data at the correct offsets. */
 #ifdef CONFIG_DETECT_HUNG_TASK
 	unsigned long			last_switch_count;
 	unsigned long			last_switch_time;
@@ ... (further down, after android_vendor_data) ...
 	struct callback_head		l1d_flush_kill;
 #endif
+#ifdef CONFIG_SYSVIPC
+	struct sysv_sem			sysvsem;	/* was ANDROID_KABI_RESERVE(1): 8 bytes */
+	struct sysv_shm			sysvshm;	/* was ANDROID_KABI_RESERVE(2),(3): list_head 16 bytes */
+#else
 	ANDROID_KABI_RESERVE(1);
 	ANDROID_KABI_RESERVE(2);
 	ANDROID_KABI_RESERVE(3);
+#endif
 	ANDROID_KABI_RESERVE(4);
 	ANDROID_KABI_RESERVE(5);
 	ANDROID_KABI_RESERVE(6);
```
Result, **pahole-verified**: `task_struct` with `SYSVIPC=on` + this patch is byte-identical to a
clean `SYSVIPC=off` build — `android_vendor_data1` sits at the same offset (**3320**) either way.
`sched_walt` and every other stock module reading `android_vendor_data*` sees exactly the layout
it expects. This is the general pattern for any *other* stock module that turns out to crash on a
future container-config addition: find the struct it reads, and relocate the new field(s) into
`ANDROID_KABI_RESERVE` slots the same way — it's GKI's own designed mechanism for exactly this.

### Supporting pieces (needed for the namespaces to actually build/link, not part of the ABI wall)

- **IPC namespace symbol exports** (`patches/kernel/ipc-exports.patch`) — `CONFIG_IPC_NS` makes
  `rust_binder.ko` reference `put_ipc_ns`/`init_ipc_ns`, which ACK's upstream source leaves
  un-exported:
  ```c
  // ipc/namespace.c
  +EXPORT_SYMBOL(put_ipc_ns);
  // ipc/msgutil.c
  +EXPORT_SYMBOL(init_ipc_ns);
  ```
  (`CONFIG_USER_NS`'s `from_kuid_munged` is already `EXPORT_SYMBOL`'d upstream, so it needs no
  patch — but it does need `USER_NS` on the **base** kernel, not just a device fragment; see next
  point.)
- **The container fragment attaches to the *base* GKI target, not a Moto device fragment**
  (`patches/kernel/lindroid_gki.fragment`, the plain `CONFIG_x=y` deltas). These are ABI-affecting
  core-kernel options — e.g. `CONFIG_USER_NS` turns `from_kuid_munged()` from a `static inline`
  into a real `EXPORT_SYMBOL`. The device build uses `//common:kernel_aarch64` as its
  `base_kernel`, and vendor DDK modules (`msm_sysstats`, etc.) link against the **base**
  `Module.symvers` — so if the option only lived in a device fragment, those modules would fail
  modpost (`"from_kuid_munged" [msm_sysstats.ko] undefined!`). The fragment is attached via
  `pre_defconfig_fragments` on `BUILD.bazel`'s `kernel_aarch64` target, with `check_defconfig =
  "disabled"` (otherwise `savedefconfig` rejects the added options):
  ```
  common_kernel(
      name = "kernel_aarch64",
  +   check_defconfig = "disabled",
  +   pre_defconfig_fragments = ["arch/arm64/configs/lindroid_gki.fragment"],
      ...
  ```
- **EVDI virtual-display driver, built in** (`CONFIG_DRM_LINDROID_EVDI=y`) — vendored under
  `drivers/gpu/drm/lindroid/` (from
  [`Linux-on-droid/lindroid-drm-loopback`](https://github.com/Linux-on-droid/lindroid-drm-loopback),
  pinned commit), hooked into the DRM `Kconfig`/`Makefile`:
  ```
  # drivers/gpu/drm/Kconfig
  +source "drivers/gpu/drm/lindroid/Kconfig"
  # drivers/gpu/drm/Makefile
  +obj-$(CONFIG_DRM_LINDROID_EVDI) += lindroid/
  ```
  Built statically (`=y`, not a module) specifically to sidestep GKI module-list checks and
  module packaging for a driver with no stock counterpart. It also carries three small 6.12
  compat fixes (`DRM_UNLOCKED` shim, `void`-returning `platform_driver::remove`, and — the
  one that actually matters for a visible desktop — `.fop_flags = FOP_UNSIGNED_OFFSET`,
  without which every `open()` of an EVDI card node fails `-EINVAL` and the Lindroid desktop is a
  black screen). See `docs/LINDROID.md` in the kernel-build repo for the full compat-patch
  writeup.

### Reproducing the exact stock vermagic string

Even with Trick 1 force-loading past CRC mismatches, module loading has a **separate**, earlier
gate: the module's embedded vermagic string must match the running kernel's (`same_magic()`),
independent of per-symbol CRCs. A plain `git describe` of this patched, dirty tree would produce
`…-g950637f7a171-maybe-dirty`, which doesn't match the factory string
`6.12.38-android16-5-g1d46253471dd-ab15048002-4k`. The build reproduces the **stock** string
instead:
- `KLEAF_USE_KLEAF_LOCALVERSION=true` (env var, exported before the bazel invocation).
- `--config=stamp` on the `bazel`/`build_with_bazel.py` invocation — this maps (via
  `build/kernel/kleaf/bazelrc/stamp.bazelrc`: `build:stamp
  --//build/kernel/kleaf:config_stamp`) to Kleaf's stamping feature, which runs
  `build/kernel/kleaf/workspace_status_stamp.py` to inject a `STABLE_SCMVERSION` into the build
  instead of leaving it blank (blank ⇒ banner regresses to `…-maybe-dirty`).
- The pinned SCM string itself — `-g1d46253471dd-ab15048002` (Moto's real commit + build number,
  not this tree's actual dirty-patched commit) — is supplied as the stamped value, so the final
  banner is byte-identical to the shipped kernel: `6.12.38-android16-5-g1d46253471dd-ab15048002-4k`.

  `scripts/build.sh` prints this banner after every build so it's a hard gate:
  ```bash
  strings "$DIST/Image" | grep -m1 '6\.12\.38-android16'
  # must read exactly: 6.12.38-android16-5-g1d46253471dd-ab15048002-4k
  ```

## Reproduce from source

Everything below lives in
[razr-fold-2026-kernel-build](https://github.com/zorrobyte/razr-fold-2026-kernel-build); its
`docs/LINDROID.md` is the authoritative writeup this page is aligned with.

```bash
# x86-64 Linux / WSL2 Ubuntu (or scripts/mac-build.sh on Apple Silicon via Rosetta)
scripts/bootstrap.sh        # assembles ~/kp-canoe/kernel_platform at the pinned Moto tag
                             # (MMI-W3WB36.36-48-5) + AOSP GKI base (~60 GB sync)
scripts/build.sh perf       # LINDROID=1 by default — builds the full Lindroid kernel
                             # (set LINDROID=0 for a plain audio-only kernel instead)
```

`scripts/build.sh` does all of this automatically and **idempotently** (a fresh `bootstrap.sh`
resets `common/`, and the next `build.sh` re-applies everything below):

1. Copies the vendored EVDI driver into `common/drivers/gpu/drm/lindroid/`, hooks `Kconfig`/
   `Makefile`.
2. Copies `configs/lindroid_gki.fragment` into
   `common/arch/arm64/configs/lindroid_gki.fragment`, patches `common/BUILD.bazel` to attach it
   (`pre_defconfig_fragments`) and set `check_defconfig = "disabled"`.
3. Appends the two `EXPORT_SYMBOL` lines to `ipc/namespace.c` / `ipc/msgutil.c` (marker-guarded).
4. Patches `kernel/module/version.c`'s `check_version()` bad-version path to force-load
   (marker-guarded `sed`).
5. Applies `lindroid/patches/task_struct-sysvipc-kabi.patch` to `include/linux/sched.h`
   (marker-guarded `patch -p1`).
6. Generates `soc-repo/moto_product.bzl` and the `moto_{perf,consolidate}_config.bzl` fragments
   (Moto's own product-config flow), then runs the actual Bazel/Kleaf build with
   `KLEAF_USE_KLEAF_LOCALVERSION=true` and `--config=stamp` (see previous section), plus
   `--notrim` (needed independently, so GKI's `MODULE_SIG_PROTECT` doesn't reject the
   stock-signed `rfkill`/`bluetooth` modules — see the harness README's finding #3).

Output: `out/msm-kernel-canoe-perf/dist/` (`Image`, `*.dtb`/`*.dtbo`, `vendor_dlkm.img` — **do
not flash the from-source `vendor_dlkm.img`**, it's incomplete). Gate checks before flashing:

```bash
strings out/msm-kernel-canoe-perf/dist/Image | grep -m1 '6\.12\.38-android16'
#  -> 6.12.38-android16-5-g1d46253471dd-ab15048002-4k   (vermagic stamp worked)
grep -w vendor_data_pad out/msm-kernel-canoe-perf/dist/*/Module.symvers
#  -> 0xf54e5881   (the one legitimate, accounted-for CRC delta from stock; see harness README)
```

Repack `Image` into a header-v4, kernel-only `boot.img` (generic ramdisk lives in `init_boot` on
this device/Android version, vendor ramdisk in `vendor_boot` — `boot.img` itself carries no
ramdisk here), then flash per the recipe above.

## Verify (on the running device, with `su`)

```bash
adb shell getprop sys.boot_completed                                   # 1
adb shell su -c 'zcat /proc/config.gz | grep -E "SYSVIPC|_NS=|DRM_LINDROID_EVDI"'
adb shell su -c 'unshare --user --pid --ipc --uts --fork echo OK'      # OK
adb shell su -c 'cat /sys/module/evdi/version; ls /dev/dri'            # 1.0.0 ; card0 renderD128
adb shell su -c 'cat /proc/asound/cards'                                # alorqrdsndcard
adb shell su -c 'dmesg | grep -iE "disagrees|Unknown symbol"'          # should be empty/warn-only
```

## Files in this repo

- `patches/kernel/version.c.patch` — Trick 1: force-load stock modules past the CRC shift.
- `patches/kernel/sched.h-kabi.patch` — Trick 2: relocate `sysvsem`/`sysvshm` into
  `ANDROID_KABI_RESERVE` slots to preserve `task_struct`'s byte layout.
- `patches/kernel/ipc-exports.patch` — `EXPORT_SYMBOL(put_ipc_ns)` / `EXPORT_SYMBOL(init_ipc_ns)`
  for `rust_binder.ko`.
- `patches/kernel/lindroid_gki.fragment` — the base-kernel defconfig fragment (namespaces,
  cgroups, squashfs, EVDI).

The prior `patches/kernel-core-changes.patch` and `patches/kernel-gki_defconfig.patch` (from the
earlier, incorrect diagnosis session — different EVDI driver path, unconditional
`check_version()` bypass, no `task_struct` KABI fix) have been removed; the four files above are
the actual, boot-verified patch set.
