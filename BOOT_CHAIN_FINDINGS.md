# Cubot KingKong ES3 Boot Chain Findings

Final research report for the Cubot KingKong ES3 active slot A boot-chain and root-state analysis.

## Executive Summary

The Cubot KingKong ES3 is rooted in the practical Magisk/userland sense, but it is not currently running a custom kernel.

The active slot A boot chain is mostly stock:

| Component | Live state | Stock match | Root relevance |
|---|---|---:|---|
| `boot_a` | Stock PAC `boot-gki.img` | yes | Not carrying root or custom kernel |
| `init_boot_a` | Magisk-patched | no | Primary root carrier |
| `vendor_boot_a` | Stock PAC `vendor_boot.img` | yes | Stock DTB, bootconfig, fstab, vendor ramdisk, modules |
| `dtbo_a` | Stock PAC `dtbo.img` | yes | Stock overlays and UFS/regulator references |
| `vbmeta_a` | One-byte flags modification | no | Descriptors match stock; signature no longer verifies |
| LK bootloader state | Unlocked/orange | n/a | Allows modified boot-chain content to boot |

The current root chain is best described as:

```text
stock boot_a
+ stock vendor_boot_a
+ stock dtbo_a
+ patched init_boot_a
+ modified vbmeta_a flags byte
+ unlocked/orange LK bootloader behavior
= Magisk root on the stock kernel
```

This result separates two questions that were previously easy to mix together:

1. The phone is rooted.
2. The phone is not currently running the custom kernel candidate.

Custom kernel work remains a separate compatibility problem, most likely involving the stock vendor boot environment, module ABI compatibility, UFS bring-up, and SC2730 regulator dependencies.

## Device and Build Context

| Field | Value |
|---|---|
| Device | Cubot KingKong ES3 |
| SoC family | Unisoc T615 / ums9230-family |
| Android build | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Running kernel release | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Active slot | `_a` |
| Storage type | UFS |
| UFS confirmation | `sprdoot.flash=ufs` |
| Active slot dump | `../slot_a_ordered/` |
| Stock PAC extraction | `/home/kali/pacextractor/pac_out/` |
| Custom candidates | `/home/kali/Kernels/CANDIDATE/` |

Confirmed active slot map:

| Partition | Block device |
|---|---|
| `boot_a` | `/dev/block/sda36` |
| `vendor_boot_a` | `/dev/block/sda38` |
| `init_boot_a` | `/dev/block/sda40` |
| `dtbo_a` | `/dev/block/sda44` |
| `vbmeta_a` | `/dev/block/sda49` |

## Boot Chain Diagram

```text
lk-sign.bin
  |
  v
LK reads unlock flag, secure state, and RPMB-backed state
  |
  v
LOCK FLAG IS : UNLOCK
androidboot.verifiedbootstate=orange
  |
  v
LK skips or tolerates verification failure in unlocked/orange state
  |
  v
vbmeta_a
  - stock descriptor set
  - modified top-level flags byte
  - signature check fails under avbtool
  |
  +--> boot_a
  |     - stock PAC boot-gki.img
  |     - stock kernel
  |
  +--> vendor_boot_a
  |     - stock DTB
  |     - stock bootconfig
  |     - stock vendor ramdisk
  |     - stock first-stage fstab
  |     - stock vendor modules
  |
  +--> dtbo_a
  |     - stock DTBO overlays
  |
  +--> init_boot_a
        - Magisk-patched ramdisk
        - Magisk overlay and backup files
        - root source
```

## Component Conclusions

### boot_a

Proven facts:

- Live active `boot_a` is byte-for-byte identical to stock PAC `boot-gki.img`.
- The live active boot image is not the custom candidate.
- The live active boot image is not the custom Magisk candidate.
- The live boot ramdisk is empty, matching stock.

Key hashes:

| Payload | SHA-256 |
|---|---|
| Stock/live unpacked kernel | `8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532` |
| Custom/custom+Magisk unpacked kernel | `773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a` |

Conclusion: the phone is currently running the stock kernel. Root is not coming from `boot_a`, and the custom kernel candidates are not active in the analyzed slot A boot path.

### init_boot_a

Proven facts:

- Live `init_boot_a` differs from stock PAC `init_boot.img`.
- Live `init_boot_a` is Magisk-patched.
- Stock and live use boot image header version 4.
- Both have an empty kernel field.
- The live ramdisk differs from stock and contains Magisk artifacts.

Magisk evidence found in live `init_boot_a`:

```text
.backup/.magisk
.backup/init.xz
.backup/.rmlist
overlay.d/sbin/magisk.xz
overlay.d/sbin/init-ld.xz
overlay.d/sbin/stub.xz
```

Conclusion: root is coming from patched `init_boot_a`, not from `boot_a`.

### vendor_boot_a

Proven facts:

- Live `vendor_boot_a` is byte-for-byte identical to stock PAC `vendor_boot.img`.
- DTB hash matches stock.
- Bootconfig matches stock:

```text
androidboot.hardware=ums9230_6h10
```

- Vendor ramdisk hash matches stock.
- `first_stage_ramdisk/fstab.ums9230_6h10` matches stock.
- Vendor module count matches stock: 157 modules.
- Boot-critical module hashes match stock.

Boot-critical files and modules checked:

| File/module | Result |
|---|---|
| `first_stage_ramdisk/fstab.ums9230_6h10` | stock match |
| `ufs_sprd.ko` | stock match |
| `sc2730-regulator.ko` | stock match |
| `sprd-pmic-spi.ko` | stock match |
| `printk_cpuid.ko` | stock match |
| `rpmb.ko` | stock match |
| `regmap-hook.ko` | stock match |

Conclusion: rooting did not modify live `vendor_boot_a`. The stock vendor boot environment remains boot-critical for future custom kernel work.

### dtbo_a

Proven facts:

- Live `dtbo_a` is byte-for-byte identical to stock PAC `dtbo.img`.
- DTBO table metadata matches stock.
- Entry count matches stock: 13 entries.
- All 13 extracted DTBO entry hashes match stock.
- All 13 decompiled DTS hashes match stock.
- UFS/regulator evidence matches stock.

DTBO confirms overlay references to:

```text
20200000.ufs
UFS modem partition paths
regulator-name = "vddvbus"
regulator-always-on
avdd12-supply
vddwcn
pmic_regulator
```

Direct strings not found in the decompiled DTBO overlays:

```text
sc2730
vddgen0
avdd18
vddcore
vddmodem
```

Those strings were found in the stock/live vendor_boot base DTB evidence, not in the DTBO overlays.

Conclusion: rooting did not modify live `dtbo_a`. DTBO supports the UFS/regulator/device-tree compatibility risk model for custom kernel work.

### vbmeta_a

Proven facts:

- Live `vbmeta_a` is not byte-for-byte stock.
- It differs from stock by exactly one byte.
- The changed byte is in the signed AVB header flags field.
- Stock reports `Flags: 0`.
- Live reports `Flags: 33554432`.
- All descriptors, keys, rollback metadata, release string, and build properties match stock.
- Live `vbmeta_a` fails `avbtool` signature verification because the signed flags field was modified.

Conclusion: live `vbmeta_a` is modified, but its partition identity map is still stock. The modification is compatible with the unlocked/orange LK behavior observed in bootloader strings.

## LK Warning Origin

The bootloader warning comes from LK-stage bootloader files, not Android userspace, Magisk, `init_boot`, or the Linux kernel.

Relevant files:

```text
/home/kali/pacextractor/pac_out/lk-sign.bin
/home/kali/pacextractor/pac_out/lk-fdl2-sign.bin
```

Relevant strings found:

```text
INFO: Press power button to pause.
INFO: Press power button to continue.
INFO: LOCK FLAG IS : LOCK!!!
INFO: LOCK FLAG IS : UNLOCK!!!
WARNNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY!!!
bootloader has been unlocked
androidboot.verifiedbootstate=orange
read_is_device_unlocked() get DeviceStatus 0x%x, *out_is_unlocked is %d.
%s: rpmb read lock flag fail! ret %d
%s: rpmb write lock flag fail! ret %d
%s: Digest does not exist, device unlocked so not initializing digest.
```

SPL files did not show the relevant warning strings:

```text
u-boot-spl-16k-ufs-sign.bin
u-boot-spl-16k-sign.bin
u-boot-spl-16k-emmc-sign.bin
```

Interpretation: the visible bootloader warning is tied to LK unlock-state handling and AVB verification behavior.

## Root-State Explanation

The active device state is:

- Rooted with Magisk and NetHunter Lite.
- Running the stock kernel from `boot_a`.
- Carrying Magisk modifications in `init_boot_a`.
- Using stock `vendor_boot_a` and stock `dtbo_a`.
- Using a modified top-level `vbmeta_a` with stock descriptors but a changed flags byte.
- Booting under unlocked/orange LK behavior.

This means root is real, but it does not prove custom kernel compatibility.

## Custom Kernel Failure Theory

The earlier custom boot failure is still best treated as a compatibility problem, not a root-state problem.

Plausible failure chain:

```text
custom boot kernel
  |
  v
stock vendor_boot modules, fstab, base DTB, and bootconfig
  |
  v
stock dtbo overlays
  |
  v
module ABI or UFS/regulator dependency mismatch
  |
  v
UFS first-stage mount failure or missing /dev/block/sda*
  |
  v
first-stage init exits
  |
  v
Attempted to kill init! exitcode=0x00007f00
```

Specific risk factors:

- `CONFIG_SCSI_UFS_SPRD=y` may make UFS built-in.
- `CONFIG_REGULATOR_SC2730` was previously suspected to be disabled in the custom kernel.
- The UFS DT node depends on SC2730 regulator rails:
  - `vcc`
  - `vddgen0`
  - `avdd12`
  - `avdd18`
  - `vddcore`
  - `vddmodem`
- Vendor modules use module versioning metadata.
- `CONFIG_MODVERSIONS=y` can make symbol CRC mismatch fatal for module loading.
- Prior module loading failure around `printk_cpuid.ko` is consistent with a `module_layout` CRC mismatch.

This theory is not proven as a runtime trace, but it is strongly supported as a compatibility risk by the stock vendor_boot and DTBO evidence.

## Next Main Quest

The boot-chain identity question is now answered. The next phase should focus on building a live Android 5.15 kernel module/header environment for the running stock kernel:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

The phone exposes:

```text
/proc/config.gz
/sys/kernel/kheaders.tar.xz
```

The next phase should use those live sources to:

- stage the kernel headers
- recover and normalize the live `.config`
- check `CONFIG_MODVERSIONS`
- determine whether usable symbol CRC data is available
- build a minimal hello-world module
- compare vermagic against the running kernel
- test whether module ABI compatibility is realistic
- evaluate whether external USB Wi-Fi, Bluetooth, or HID modules are feasible

Detailed plan: `NEXT_PHASE_HEADERS_MODULES_PLAN.md`.

Do not treat this report as a flashing instruction.
