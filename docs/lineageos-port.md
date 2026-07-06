# LineageOS 23.2 port — milestone-1 strategy, flash recipe, hardware survey

Full detail and reproducible build steps live in the device tree repo,
[android_device_motorola_blanc](https://github.com/zorrobyte/android_device_motorola_blanc)
(branch `lineage-23.2`). This page summarizes the strategy and the parts relevant to the
Lindroid bring-up; see that repo's README for the up-to-date checklist.

## Milestone-1 strategy: Lineage system on stock everything-else

Only `system`, `system_ext`, and `product` are built and flashed from LineageOS.
**Kernel, `vendor`, `vendor_dlkm`, `odm`, `boot`, `init_boot`, `vendor_boot`, `dtbo` all stay
stock.** This means:

- The entire Qualcomm/Moto HAL stack is untouched, and — critically — the
  **stock-kernel↔stock-`vendor_dlkm`-modules pairing is preserved**. This is the same
  CRC-matching concern that later bites the from-source Lindroid kernel (see
  [lindroid-kernel.md](lindroid-kernel.md)): kernel and `vendor_dlkm` must come from the same
  build, or module version-CRC checks fail and boot dies.
- AVB (Android Verified Boot) is disabled by flag-patching the factory `vbmeta`, rather than
  by wiping the whole verified-boot chain. `fastboot` 37.0.0 rejects `--disable-verity` on
  Moto's `vbmeta` image ("Failed to find AVB_MAGIC") even though the image is valid, so the
  device tree's `scripts/patch-vbmeta.py` patches the AVB flags byte directly (sets `flags=3`
  = verity + verification both disabled) and that patched image is flashed instead.
- Rollback is simple: reflash stock `system`/`system_ext`/`product` from the unpacked factory
  `super.img` (`simg2img` + `lpunpack`).

This bought a working device on the **first flash attempt**, with SELinux enforcing, without
first solving blob extraction, sepolicy, or a custom kernel. Blob extraction (a
self-contained vendor, decoupled from stock) and the from-source kernel are later phases —
the kernel phase is exactly what [razr-fold-2026-kernel-build](https://github.com/zorrobyte/razr-fold-2026-kernel-build)
and, further, the Lindroid kernel work in this repo build toward.

## Flash recipe (verified)

```bash
# bootloader unlocked; THIS WIPES DATA (stock encryption keys don't survive)
adb reboot bootloader

# fastboot chokes on Moto's vbmeta with --disable-verity ("Failed to find AVB_MAGIC");
# patch the flags byte directly instead of relying on fastboot's verity toggle.
python3 scripts/patch-vbmeta.py vbmeta.img vbmeta-disabled.img   # sets flags=3
fastboot flash vbmeta vbmeta-disabled.img
fastboot --disable-verity --disable-verification flash vbmeta_system vbmeta_system.img

fastboot reboot fastboot          # userspace fastboot, needed for dynamic partitions
fastboot flash system system.img
fastboot flash system_ext system_ext.img
fastboot flash product product.img
fastboot -w                       # mandatory data wipe
fastboot reboot
```

Recovery from a bad flash: reflash the three stock images extracted from the factory `super`
(`simg2img super.img_sparsechunk.* super.raw && lpunpack -p system_a,system_ext_a,product_a super.raw`).
Magisk root (kept in stock `init_boot`, untouched by this recipe) survives a data wipe; just
reinstall the Magisk app to let it repair `/data`.

## Hardware survey (verified working after first flash)

| Subsystem | Status |
|---|---|
| Display / touch / UI | working |
| Wi-Fi | working |
| RIL | working |
| NFC | working |
| Bluetooth | working |
| Sensors, incl. Moto `hinge_posture` | working |
| GPS provider | working |
| Audio HALs | working |
| Fingerprint (goodix HAL) | running |
| Rear camera | working |
| Inner selfie camera | working |
| Cover-panel selfie camera (folded) | fixed this session — see [fixes.md](fixes.md) |
| Cover-panel 165 Hz refresh | fixed this session — see [fixes.md](fixes.md) |

## Device facts

| | |
|---|---|
| Codename | `blanc` (product `blanc_gu`, SKU XT2651-2) |
| SoC | Qualcomm SM8845 (`canoe`), 4 KB pages |
| Android | 16 (SDK 36), board API 202504 |
| Stock build | `W3WBS36.36-48-5-1` |
| Stock kernel | GKI `6.12.38-android16-5` |
| Displays | Inner 2232×2484 up to 120 Hz; cover 1080×2520 up to 165 Hz; both with camera cutouts |
| Partitions | A/B, dynamic `super` 28.8 GB (erofs), dedicated recovery |
| Device states | `0`=CLOSED `1`=TENT `2`=STAND `3`=LAPTOP `4`=OPENED `5`=REAR_DISPLAY_MODE `6`=CONCURRENT_INNER_DEFAULT `7`=REAR_DISPLAY_OUTER_DEFAULT `8`=HALF_OPENED_INNER |

The device-state table above is read from the stock
`/vendor/etc/devicestate/device_state_configuration.xml` and matters directly for the
folded-camera fix and the fold-state device-state arrays populated in the device tree overlay
(see [fixes.md](fixes.md)).
