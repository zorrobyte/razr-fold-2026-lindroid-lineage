# Lindroid kernel — reuse, SYSVIPC fix, boot panic, diagnosis

This is the current blocker for the whole Lindroid-on-Lineage effort. Everything below is
from this session's build tree (`common` under the Lindroid kernel workspace) and the actual
flash attempt.

## What kernel this is

Rather than building a fresh Lindroid kernel from scratch, this reuses the prebuilt Lindroid
kernel from the earlier stock effort's build tree, and fixes it forward:

- `CONFIG_SYSVIPC`, `CONFIG_IPC_NS`, `CONFIG_POSIX_MQUEUE` were **clobbered to `n`** in that
  prebuilt's defconfig (needed for LXC namespaces / SysV IPC inside the container) — re-enabled.
- Rebuilt with Bazel's `--config=stamp`, because without it the kernel's version string
  regresses from a real git-describe to `…-maybe-dirty`, which matters for vermagic matching
  against the factory `vendor_dlkm`.
- Resulting `Image` banner: `6.12.38-android16-5-g1d46253471dd-ab15048002-4k` — **matches
  stock** exactly.
- `.config` confirmed post-build:

  ```
  CONFIG_SYSVIPC=y
  CONFIG_POSIX_MQUEUE=y
  CONFIG_UTS_NS=y
  CONFIG_IPC_NS=y
  CONFIG_USER_NS=y
  CONFIG_PID_NS=y
  CONFIG_NET_NS=y
  CONFIG_DRM_LINDROID_EVDI=y
  ```

## The kernel-side Lindroid deltas (from `git diff` in the kernel `common` tree)

| File | What changed |
|---|---|
| `arch/arm64/configs/gki_defconfig` | SYSVIPC/IPC_NS/POSIX_MQUEUE/EVDI enabled (see [`patches/kernel-gki_defconfig.patch`](../patches/kernel-gki_defconfig.patch) — see note below on why this diff is huge) |
| `drivers/gpu/drm/Kconfig`, `drivers/gpu/drm/Makefile` | wire in `drivers/gpu/drm/evdi/` as `CONFIG_DRM_LINDROID_EVDI` |
| `include/drm/drm_ioctl.h` | add the `DRM_UNLOCKED` ioctl flag bit used by the EVDI driver |
| `ipc/msgutil.c`, `ipc/namespace.c` | `EXPORT_SYMBOL_GPL(init_ipc_ns)`, `EXPORT_SYMBOL_GPL(put_ipc_ns)` — needed by `rust_binder.ko` |
| `kernel/module/version.c` | **`check_version()` neutered to unconditionally `return 1`** — see below, this matters a lot for the diagnosis |
| `BUILD.bazel` | `check_defconfig = "disabled"`, `kmi_enforced = False`, `kmi_symbol_list_strict_mode = False` |
| `build.config.gki` | drops the `check_defconfig` post-defconfig command |

All of the above (except the giant defconfig diff) is in
[`patches/kernel-core-changes.patch`](../patches/kernel-core-changes.patch).

**Note on `kernel-gki_defconfig.patch`:** the actual `git diff` for `gki_defconfig` is ~8,900
lines, because the working tree's defconfig has been fully expanded/materialized (every
default-value symbol spelled out) rather than staying a minimal defconfig fragment — most of
that diff is reformatting, not intentional changes. The functionally relevant lines are just
the SYSVIPC/IPC_NS/POSIX_MQUEUE/EVDI ones shown above; the patch is included for completeness
but should not be read hunk-by-hunk as "the fix."

### `check_version()` neutering — read this before trusting any CRC-mismatch theory

```c
int check_version(const struct load_info *info,
                   const char *symname,
                   struct module *mod,
                   const u32 *crc)
{
    return 1; /* lindroid: bypass module CRC; GKI ABI stable */
}
```

This function is the kernel's runtime MODVERSIONS gate — it's what normally rejects a `.ko`
module and logs `"%s: disagrees about version of symbol %s"` when a module's recorded symbol
CRC doesn't match the running kernel's. It has been **hard-disabled** here (always succeeds),
alongside the Bazel-level `kmi_enforced = False` / `kmi_symbol_list_strict_mode = False` /
`check_defconfig = "disabled"` (those three are build-time GKI-ABI-drift guards, unrelated to
this runtime check, but show the same intent: stop the build/load pipeline from blocking on
ABI-list mismatches introduced by the Lindroid config deltas).

This directly affects the diagnosis below: **if this kernel already had `check_version()`
neutered at flash time** (very likely — build logs from the prior SYSVIPC-fix iteration show
an earlier attempt failing verification with `MODVERSIONS-still-on`, followed by attempts that
succeeded, consistent with this patch being what fixed that), **then a clean CRC-mismatch
rejection is not what killed boot** — that specific failure mode is bypassed by design. See
"Open questions" below.

## Packaging and flash

- Repacked as `boot.img` **header v4, kernel-only** (`--header_version 4 --kernel
  bootunpack/kernel --ramdisk bootunpack/ramdisk`, with the ramdisk slot pointed at a 0-byte
  placeholder). This is correct for this device/Android version: the generic ramdisk lives in
  `init_boot` (A13+) and the vendor ramdisk lives in `vendor_boot` — `boot.img` itself is
  kernel-only.
- Flashed to the device (`fastboot flash boot …`).

## Result: boot failure

The device **did not boot**. `pstore`/`ramoops` captured that the kernel boots fine and
reaches `init` (PID 1) at approximately **0.79 s**, then:

```
Kernel panic - not syncing: Attempted to kill init! exitcode=0x00007f00
```

(`WEXITSTATUS` 127) — a QCOM watchdog subsequently bit. This is the standard Android panic
raised when PID 1 itself dies; unlike a driver-probe failure, it does not by itself name a
specific module or subsystem.

**Recovery (clean, ~30 s):** only the `boot` partition was touched, so recovery was
reflashing the stock `boot` image. A failed-boot A/B slot switch to `_b` occurred as expected;
fixed with `fastboot set_active a` + reflashing stock `boot` to `boot_a`.

## Diagnosis

**Ruled out:** a filesystem-mount issue. The Lindroid kernel's `.config` matches stock on
`EROFS_FS_ZIP=y`, F2FS, `DM_VERITY`, overlayfs, and inline-crypt — none of the mount-relevant
config was touched, and `init_boot`/`vendor_boot`/`vendor_dlkm`/`system` are all stock (which
boot fine paired with the stock kernel).

**Leading theory (as of the panic, before the `check_version` finding above):** vendor module
incompatibility. This is a GKI kernel; the Lindroid config deltas (namespaces, EVDI, SYSVIPC)
shift `struct` layouts and therefore module symbol CRCs, even though the version **string**
was made to match stock. So stock `vendor_dlkm` modules would fail their MODVERSIONS check
against this kernel even though the vermagic *string* matches — and if a driver a critical
`init`-path service depends on fails to load, `init` can die. This is the same class of failure
this project's own kernel-build notes warn about generally ("never mix a from-source kernel
with factory `vendor_dlkm` → CRC bootloop"); the vermagic-string-match trick was meant to dodge
it but had never actually been boot-tested until this attempt.

**Complication found this session, documented above:** `check_version()` — the exact function
that would raise a MODVERSIONS rejection — is hard-neutered to always succeed in this kernel's
source tree, and that neutering was very likely already present in the build that was flashed
and panicked (see the attempt-log evidence above). If so, a *clean* "disagrees about version of
symbol" rejection is not the mechanism here; that door was already closed on purpose. That
leaves a few sharper possibilities the next attempt should specifically look for:

1. **A genuinely missing/renamed symbol** (an "Unknown symbol" load failure, not a version
   mismatch) — the CRC bypass only helps when the symbol name still exists; if the struct
   layout change removed or renamed an export, `check_version` is never reached for it and the
   module fails to load outright.
2. **A module that loads but misbehaves** — with CRC checking off, an ABI-incompatible module
   *can* now load, read/write through a mismatched `struct` layout, and corrupt state or crash
   somewhere else at runtime (a worse and harder-to-diagnose failure than a clean rejection).
3. **Something unrelated to kernel modules at all** — `init` dying with exit status 127 can
   also indicate a completely different early-boot dependency problem (missing/incompatible
   shared library, `ueventd`/`init.rc` service failure cascading up). The pstore capture wasn't
   fully combed for "Unknown symbol", `modprobe`/`insmod` failures, or `init.rc` service errors
   beyond the panic line itself.

## Next steps

- **(a)** Build a matched `vendor_dlkm` from the *same* kernel source tree, so there's no
  struct-layout delta between kernel and vendor modules at all (removes the whole class of
  problem, whether or not CRC checking is enforced).
- **(b)** On the next flash attempt, capture the **full** `console-ramoops` (not just the
  panic line) and grep specifically for `Unknown symbol`, `insmod`/`init_module` return codes,
  and any `init.rc` service crash-loop, to distinguish the three possibilities above — since
  the leading "disagrees about version of symbol" signature is now known to be suppressed by
  the `check_version` patch and would not appear even if a CRC mismatch is technically present.
- **(c)** Alternatively, rebuild the kernel minimizing the config delta from stock (i.e., only
  the bare minimum needed for SYSVIPC/namespaces/EVDI) specifically to preserve struct layouts
  and thus real CRC compatibility with the stock `vendor_dlkm` — making option (a) unnecessary.
- Re-examine whether `check_version()` should be **re-enabled** for the next diagnostic flash
  (accepting the bootloop-on-real-mismatch risk) specifically *in order to* get a clean
  "disagrees about version of symbol NAME" log line that identifies the exact offending module
  and symbol — trading a worse failure mode for a much more actionable error message.
