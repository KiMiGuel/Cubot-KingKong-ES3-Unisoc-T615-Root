# Vendor Boot Stock vs Live Comparison

Scope: `vendor_boot` only. No original images were modified. No flashing, patching, repacking, resigning, or new boot image generation was performed. Extracted artifacts were written under `./comparison/vendor_boot_stock_vs_live/`.

## A. Executive Summary

Live active slot A `vendor_boot_a` is an exact byte-for-byte match for stock PAC `vendor_boot.img`.

The normalized payloads also match:

- DTB hash matches.
- bootconfig hash and content match.
- vendor ramdisk hash matches.
- first-stage fstab hash matches.
- module file list and module count match.
- boot-critical UFS, SC2730 regulator, PMIC, RPMB, and early-loaded module hashes match.

Conclusion: there is no evidence that live `vendor_boot_a` was modified during rooting. The live root evidence remains in `init_boot_a`, not `boot_a` or `vendor_boot_a`.

## B. Side-by-Side Table

| Field | Stock vendor_boot | Live vendor_boot_a |
|---|---:|---:|
| Source path | `/home/kali/pacextractor/pac_out/vendor_boot.img` | `../slot_a_ordered/03_vendor_boot_a_live.img` |
| Raw size | 104,857,600 bytes | 104,857,600 bytes |
| Raw SHA256 | `0df796cca5a5a80b1d93caeca17acbcd075262ab3e2ff73bbe6e99fa075efadf` | `0df796cca5a5a80b1d93caeca17acbcd075262ab3e2ff73bbe6e99fa075efadf` |
| `file` output | data | data |
| Vendor boot header version | 4 | 4 |
| Page size | `0x00001000` | `0x00001000` |
| Vendor cmdline | `console=ttyS1,115200n8 bootconfig bootconfig` | `console=ttyS1,115200n8 bootconfig bootconfig` |
| Bootconfig size | 34 bytes | 34 bytes |
| Bootconfig content | `androidboot.hardware=ums9230_6h10` | `androidboot.hardware=ums9230_6h10` |
| DTB size | 132,833 bytes | 132,833 bytes |
| Vendor ramdisk total size | 33,809,764 bytes | 33,809,764 bytes |
| Vendor ramdisk fragments | one fragment, `vendor_ramdisk00`, type `0x1` | one fragment, `vendor_ramdisk00`, type `0x1` |
| AVB footer | present | present |

## C. Raw Image Comparison

| Pair | Exact raw match |
|---|---:|
| Stock PAC vendor_boot vs Live active slot A vendor_boot | yes |

Raw hashes are saved in:

- `STOCK_LIVE_VENDOR_BOOT_SHA256SUMS.txt`
- `comparison/vendor_boot_stock_vs_live/RAW_IMAGE_METADATA.txt`

`avbtool info_image` also reports matching vendor_boot AVB metadata:

- Original image size: 33,959,936 bytes
- VBMeta offset: 33,959,936
- VBMeta size: 2,240 bytes
- Public key SHA1: `8b41486347306814101259310ab4e64cc5bc7c5e`
- Hash descriptor digest: `45a889bcc30d0b54d4ef86b83658e398edbaa7e5a05d9a0f8a32fd2a3cffa48a`

## D. Normalized Payload Comparison

| Field | Stock vendor_boot | Live vendor_boot_a | Match |
|---|---:|---:|---:|
| DTB SHA256 | `65dfe08c027d2c502e5a1fc3c8e2a76f22dd6e9ac8b63362f08ded9b7b929776` | `65dfe08c027d2c502e5a1fc3c8e2a76f22dd6e9ac8b63362f08ded9b7b929776` | yes |
| bootconfig SHA256 | `d5179fc1f67658a6b024ef84f5d847e83b9d33b644b7dc8cd61c290803faf166` | `d5179fc1f67658a6b024ef84f5d847e83b9d33b644b7dc8cd61c290803faf166` | yes |
| vendor ramdisk SHA256 | `3f40b369306e77b5c4223a8cdfa0fd65038bcac471273d6ce57be0db5d94f496` | `3f40b369306e77b5c4223a8cdfa0fd65038bcac471273d6ce57be0db5d94f496` | yes |
| vendor ramdisk cpio entries | 1,114 | 1,114 | yes |
| extracted regular file count | 838 | 838 | yes |
| module count | 157 | 157 | yes |
| stock-only files | 0 | n/a | yes |
| live-only files | n/a | 0 | yes |
| stock-only modules | 0 | n/a | yes |
| live-only modules | n/a | 0 | yes |
| `first_stage_ramdisk/fstab.ums9230_6h10` SHA256 | `efef71e3305a94fff71bde0f70725051e5ab8cc5c17e048beecbe89ba68fdc44` | `efef71e3305a94fff71bde0f70725051e5ab8cc5c17e048beecbe89ba68fdc44` | yes |
| `lib/modules/modules.load` SHA256 | `fff35ba1ac956b98da45d7caee5e2990053e903adfada49b4ec4ed0725d5d67a` | `fff35ba1ac956b98da45d7caee5e2990053e903adfada49b4ec4ed0725d5d67a` | yes |
| `lib/modules/modules.dep` SHA256 | `3e89ba3e5d6ba61ff74747cb85fe38750cf7e803b42a5181d74c49e554921b97` | `3e89ba3e5d6ba61ff74747cb85fe38750cf7e803b42a5181d74c49e554921b97` | yes |

First-stage active-hardware fstab entries match stock exactly. The active `fstab.ums9230_6h10` includes first-stage logical mounts for `/system`, `/system_ext`, `/vendor`, `/odm`, `/product`, `/vendor_dlkm`, `/system_dlkm`, and `/metadata`.

## E. Boot-Critical Module Table

| Module/File | Stock present | Live present | Hash match | Notes |
|---|---:|---:|---:|---|
| `first_stage_ramdisk/fstab.ums9230_6h10` | yes | yes | yes | Active-hardware first-stage fstab |
| `lib/modules/ufs_sprd.ko` | yes | yes | yes | UFS host driver; depends on `sprd_soc_id.ko`, `rpmb.ko`, `sprd_sip_svc.ko`, `sysdump.ko` |
| `lib/modules/sc2730-regulator.ko` | yes | yes | yes | SC2730 regulator module |
| `lib/modules/sprd-pmic-spi.ko` | yes | yes | yes | SC2730 PMIC SPI support; depends on `sysdump.ko` |
| `lib/modules/printk_cpuid.ko` | yes | yes | yes | Early loaded in `modules.load` |
| `lib/modules/rpmb.ko` | yes | yes | yes | Dependency for `ufs_sprd.ko` |
| `lib/modules/regmap-hook.ko` | yes | yes | yes | Early loaded in `modules.load` |

All checked critical modules have identical hashes and the same vermagic:

```text
5.15.178-android13-8-g5b7f481f7f44-dirty SMP preempt mod_unload modversions aarch64
```

## F. UFS / Regulator Evidence

Bootconfig:

```text
androidboot.hardware=ums9230_6h10
```

First-stage fstab:

```text
system /system erofs ro wait,avb=vbmeta_system,logical,first_stage_mount,avb_keys=/avb/q-gsi.avbpubkey:/avb/r-gsi.avbpubkey:/avb/s-gsi.avbpubkey,slotselect
system_ext /system_ext erofs ro wait,avb=vbmeta_system,logical,first_stage_mount,slotselect
vendor /vendor erofs ro wait,avb=vbmeta_vendor,logical,first_stage_mount,slotselect
odm /odm erofs ro wait,avb=vbmeta_odm,logical,first_stage_mount,slotselect
product /product erofs ro wait,avb=vbmeta_product,logical,first_stage_mount,slotselect
vendor_dlkm /vendor_dlkm erofs ro wait,avb=vbmeta_system_ext,logical,first_stage_mount,slotselect
system_dlkm /system_dlkm erofs ro wait,avb=vbmeta_system_ext,logical,first_stage_mount,slotselect
/dev/block/by-name/metadata /metadata f2fs nodev,noatime,nosuid wait,formattable,first_stage_mount,check
```

DTB strings include UFS and regulator evidence:

```text
ufs@20200000
/soc/ap-apb/ufs@20200000
sprd,sc2730-regulator
vdd-vddgen0-supply
vdd-avdd12-supply
vdd-avdd18-supply
vdd-vddcore-supply
vdd-vddmodem-supply
```

Vendor ramdisk UFS block-device evidence includes:

```text
/dev/block/sda[0-9]*  u:object_r:ufs_device:s0
/dev/block/sdb[0-9]*  u:object_r:ufs_device:s0
/dev/block/sdc[0-9]*  u:object_r:ufs_device:s0
```

The stock and live results are identical for all vendor_boot UFS/regulator checks. There are no stock-vs-live vendor_boot differences relevant to UFS or SC2730 regulator boot behavior.

## G. Clear Conclusion

Does live vendor_boot match stock exactly?

Yes. The stock PAC `vendor_boot.img` and live active slot A `vendor_boot_a` have identical raw SHA256 values and `cmp` reports an exact raw match.

If not exact, what differs?

Nothing was found to differ. Raw image, AVB footer metadata, DTB, bootconfig, vendor ramdisk, first-stage fstab, module list, module count, and checked boot-critical module hashes all match.

Is there any evidence vendor_boot_a was modified during rooting?

No. The live `vendor_boot_a` is byte-for-byte stock. The Magisk/root modification evidence remains in `init_boot_a`.

Does vendor_boot support the theory that custom kernel failure involved module ABI/UFS/regulator mismatch?

It supports the risk model, but not because live vendor_boot differs from stock. The stock/live vendor_boot contains UFS and regulator modules built with vermagic `5.15.178-android13-8-g5b7f481f7f44-dirty` and `CONFIG_MODVERSIONS`-style module metadata. A custom kernel with a different release or symbol CRC set could fail to load these stock vendor_boot modules, including `ufs_sprd.ko`, `sc2730-regulator.ko`, `sprd-pmic-spi.ko`, `rpmb.ko`, and related early dependencies. This report does not prove that happened at runtime, but it keeps module ABI/UFS/regulator mismatch as a plausible custom-kernel failure mechanism.

## H. Next Recommended Step

Stop after this `vendor_boot` comparison.

The next analysis step should be a controlled stock-vs-live `dtbo` comparison, because vendor_boot is stock and its DTB points to UFS/regulator dependencies that may need DTBO confirmation.

Do not treat this report as a flashing recommendation.

## Artifacts

Unpacked comparison artifacts:

```text
comparison/vendor_boot_stock_vs_live/stock_vendor_boot/
comparison/vendor_boot_stock_vs_live/live_vendor_boot_a/
```

Supporting command-output files:

```text
comparison/vendor_boot_stock_vs_live/RAW_IMAGE_METADATA.txt
comparison/vendor_boot_stock_vs_live/EXACT_MATCH_MATRIX.txt
comparison/vendor_boot_stock_vs_live/UNPACK_BOOTIMG_METADATA.txt
comparison/vendor_boot_stock_vs_live/NORMALIZED_PAYLOAD_SHA256SUMS.txt
comparison/vendor_boot_stock_vs_live/BOOTCONFIG_CONTENTS.txt
comparison/vendor_boot_stock_vs_live/AVBTOOL_VENDOR_BOOT_INFO.txt
comparison/vendor_boot_stock_vs_live/RAMDISK_AND_MODULE_COUNTS.txt
comparison/vendor_boot_stock_vs_live/FSTAB_COMPARE_SUMMARY.txt
comparison/vendor_boot_stock_vs_live/FSTAB_MODULE_CONTROL_HASHES.txt
comparison/vendor_boot_stock_vs_live/BOOT_CRITICAL_MODULE_TABLE.md
comparison/vendor_boot_stock_vs_live/BOOT_CRITICAL_SEARCH_COUNT_TABLE.md
```
