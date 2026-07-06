# Fixes made this session

Three fixes to the LineageOS frameworks, each extracted as a real `git diff` from the working
tree (`~/android/lineage/frameworks/{base,native}` in the build environment) into
[`patches/`](../patches/). None of these have been upstreamed; they live as local commits on
top of `lineage-23.2` in the build tree.

## 1. Cover display stuck at 60 Hz (panel does 165)

**Root cause:** the cover panel's HWC default mode is 60 Hz, even though the panel supports up
to 165 Hz (the inner panel defaults to 120 Hz and is unaffected). Stock handles this with
Motorola's adaptive-refresh service, which lives on the `product` partition — and LineageOS's
milestone-1 strategy replaces `product` (see [lineageos-port.md](lineageos-port.md)), so that
service is gone.

**Fix:** a device-tree overlay (`frameworks/base/core/res/res/values/config.xml`) setting
`config_defaultPeakRefreshRate=165` and `config_defaultRefreshRate=0`, so
`DisplayModeDirector` ranges up to each panel's own maximum instead of pinning the cover
display at its 60 Hz default. The same overlay populates the foldable device-state arrays
(`config_foldedDeviceStates`, `config_halfFoldedDeviceStates`, `config_openDeviceStates`,
`config_rearDisplayDeviceStates`) from the stock device-state semantics, since several
framework paths still consult these arrays even though Android 16 primarily uses the semantic
properties in `device_state_configuration.xml`.

This fix lives in the device tree repo, not this one — it's already committed at
[`android_device_motorola_blanc:overlay/frameworks/base/core/res/res/values/config.xml`](https://github.com/zorrobyte/android_device_motorola_blanc/blob/lineage-23.2/overlay/frameworks/base/core/res/res/values/config.xml).
**Verified on device:** forcing 165 switches the cover panel cleanly.

## 2. Folded selfie camera fails (`CamxResultEInvalidArg`)

**Symptom:** opening the logical front camera (camera 1) while the device is physically
folded fails in the Moto CamX HAL with `CamxResultEInvalidArg`.

**Root cause — proven empirically this session:** forcing the device-state flags to
`DEVICE_STATE_OPENED` while the device is physically folded lets camera 1 open successfully.
The Moto CamX HAL rejects the front camera specifically when it receives
`DEVICE_STATE_FOLDED` **without** an accompanying front-covered bit. AOSP's
`CameraServiceProxy` only ever sends `DEVICE_STATE_FOLDED` on fold — it never sends
`DEVICE_STATE_FRONT_COVERED` / `DEVICE_STATE_BACK_COVERED`, so the Moto HAL always sees an
"impossible" combination on this device and refuses to open the logical front camera while
folded.

**Fix:** a new gated device config, `config_cameraFrontCoveredWhenFolded` (default `false`,
added to `frameworks/base`'s `config.xml` + `symbols.xml`), and a change to
`CameraServiceProxy.java`'s `FoldStateListener` callback so that when the device folds, it ORs
in `ICameraService.DEVICE_STATE_FRONT_COVERED` alongside `DEVICE_STATE_FOLDED` **iff** the
overlay enables the config. `blanc`'s device tree overlay sets it `true`. This is intentionally
gated rather than unconditional, since AOSP's default fold semantics are correct for
non-Motorola foldables.

Patch: [`patches/camera-folded-selfie.patch`](../patches/camera-folded-selfie.patch)
(`frameworks/base`, 3 files, 21 lines).

```java
mFoldStateListener = new FoldStateListener(mContext, folded -> {
    if (folded) {
        int deviceStateFlags = ICameraService.DEVICE_STATE_FOLDED;
        // Some foldables (e.g. Motorola razr) need the inner front camera
        // reported as covered when folded, so the vendor HAL routes the
        // logical front camera to the outer/cover lens. Without this bit the
        // Moto CamX HAL rejects opening the front camera folded with
        // CamxResultEInvalidArg. Gated per-device via config overlay.
        if (mContext.getResources().getBoolean(
                R.bool.config_cameraFrontCoveredWhenFolded)) {
            deviceStateFlags |= ICameraService.DEVICE_STATE_FRONT_COVERED;
        }
        setDeviceStateFlags(deviceStateFlags);
    } else {
        clearDeviceStateFlags(ICameraService.DEVICE_STATE_FOLDED
                | ICameraService.DEVICE_STATE_FRONT_COVERED);
    }
});
```

**Status: built, not yet flash-verified.** (The device tree's published README still lists
this as "under investigation" pending a rebuild/flash to confirm.)

## 3. Lindroid input isolation — EventHub `.idc` disable

**Task:** Lindroid's container/EVDI path registers virtual input devices (from the LXC guest
and the EVDI virtual display) at the kernel `evdev` layer. Those events must not reach Android
apps as if they were real touch/key input.

**Fix:** ported Billy Laws' LMODroid change
([gerrit 12936](https://gerrit.libremobileos.com/c/LMODroid/platform_frameworks_native/+/12936))
to Android 16's `std::optional`-based `PropertyMap::getBool` API. `EventHub::openDeviceLocked`
now checks the device's `.idc` config for `device.disabled=1` and calls `device->disable()` if
set, so a virtual input device can be excluded from Android's input pipeline purely by config,
without touching `InputReader` policy or SELinux.

Patch: [`patches/inputflinger-idc-disable.patch`](../patches/inputflinger-idc-disable.patch)
(`frameworks/native/services/inputflinger/reader/EventHub.cpp`, 10 lines).

```cpp
// Lindroid: allow disabling an input device via its .idc (device.disabled=1).
// The container/EVDI path registers virtual input devices whose events must not
// reach Android apps. Ported to Android 16 from Billy Laws' inputflinger change
// (gerrit.libremobileos.com/c/LMODroid/platform_frameworks_native/+/12936).
if (device->configuration &&
    device->configuration->getBool("device.disabled").value_or(false)) {
    ALOGV("Disabling device with id %d\n", deviceId);
    device->disable();
}
```

**Why this replaces the stock effort's approach:** the stock (Magisk-based) Lindroid effort
solved analogous "the phone shouldn't see the container's internals" problems by smali-patching
the shipped `services.jar` — see its
[`docs/services-jar.md`](https://github.com/zorrobyte/razr-fold-2026-lindroid/blob/main/docs/services-jar.md)
(a `canJoinSharedUserId` patch so `org.lindroid.ui` can join `android.uid.system`, and a
`WiredAccessoryManager` null-guard for an EVDI-hotplug-triggered NPE that killed
`system_server`). Both of those exist only because stock Android can't be recompiled: patching
means baksmali/smali round-tripping, dex repacking, and whiting out the stale
odex/vdex/art cache with `mknod c 0 0` so ART picks up the patched dex — with the ever-present
risk of Magisk auto-disabling all modules after ~2 bootloops. On a self-built ROM, the
equivalent class of fix — as demonstrated by the `EventHub.cpp` change above — is just a
source change and a rebuild. No dex hacks are needed on Lineage.
