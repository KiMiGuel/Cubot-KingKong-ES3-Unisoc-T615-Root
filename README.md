# Cubot KingKong ES3 Unisoc T615 Root Research

This repository documents boot-chain, AVB, Magisk/root-state, and kernel-module feasibility research for the Cubot KingKong ES3.

It is an analysis and documentation repo. Raw firmware images, PAC files, partition dumps, and extracted proprietary firmware trees are intentionally not stored here.

## Device

| Field | Value |
|---|---|
| Device | Cubot KingKong ES3 |
| SoC | Unisoc T615 / ums9230-family |
| Android build | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Running kernel | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Active slot analyzed | slot A |
| Storage | UFS |
| UFS command-line evidence | `sprdoot.flash=ufs` |
| Root state | Magisk-rooted with NetHunter Lite |

## Current Status

The phone is rooted, but it is running the stock kernel.

The active boot chain is mostly stock:

| Partition | Finding |
|---|---|
| `boot_a` | Byte-for-byte stock PAC `boot-gki.img` |
| `init_boot_a` | Magisk-patched and carrying root |
| `vendor_boot_a` | Byte-for-byte stock PAC `vendor_boot.img` |
| `dtbo_a` | Byte-for-byte stock PAC `dtbo.img` |
| `vbmeta_a` | One-byte modified flags field; descriptors match stock |

The root source is patched `init_boot_a`, not `boot_a`.

## Rooting Model Confirmed by the Evidence

The analyzed rooted state looks like this:

```text
unlocked/orange LK bootloader
  |
  v
modified vbmeta_a flags byte, stock descriptors
  |
  v
stock boot_a kernel
  |
  v
stock vendor_boot_a and stock dtbo_a
  |
  v
Magisk-patched init_boot_a
  |
  v
Magisk root
```

This means:

- Magisk root works from `init_boot_a`.
- The active kernel is still stock.
- A rooted phone is not proof that the custom kernel candidate works.
- Future kernel work must match the live stock kernel ABI or solve the module/header environment first.

For a cleaner guide to the process and the evidence behind it, see `ROOTING_GUIDE.md`.

## Major Findings

### boot_a

Live `boot_a` is byte-for-byte identical to stock PAC `boot-gki.img`.

Stock/live unpacked kernel hash:

```text
8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532
```

Custom candidate kernel hash:

```text
773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a
```

### init_boot_a

Live `init_boot_a` is Magisk-patched. Evidence includes:

```text
.backup/.magisk
.backup/init.xz
.backup/.rmlist
overlay.d/sbin/magisk.xz
overlay.d/sbin/init-ld.xz
overlay.d/sbin/stub.xz
```

### vendor_boot_a

Live `vendor_boot_a` is stock. It contains the stock DTB, bootconfig, vendor ramdisk, first-stage fstab, and 157 stock vendor modules.

Bootconfig:

```text
androidboot.hardware=ums9230_6h10
```

Boot-critical stock-matching modules include:

```text
ufs_sprd.ko
sc2730-regulator.ko
sprd-pmic-spi.ko
printk_cpuid.ko
rpmb.ko
regmap-hook.ko
```

### dtbo_a

Live `dtbo_a` is stock. It contains 13 DTBO entries, all matching stock after extraction and decompilation.

DTBO confirms UFS and regulator-related overlay evidence, including:

```text
20200000.ufs
regulator-name = "vddvbus"
regulator-always-on
avdd12-supply
vddwcn
pmic_regulator
```

### vbmeta_a

Live `vbmeta_a` differs from stock by exactly one byte in the signed AVB header flags field.

| Field | Stock | Live |
|---|---:|---:|
| Flags | `0` | `33554432` |

All descriptors, keys, rollback metadata, release string, and build properties match stock. Live `vbmeta_a` fails `avbtool` signature verification, which fits the unlocked/orange LK bootloader state.

## Report Index

Start here:

- `BOOT_CHAIN_FINDINGS.md`
- `REPORT_INDEX.md`
- `ROOTING_GUIDE.md`
- `NEXT_PHASE_HEADERS_MODULES_PLAN.md`

Component reports:

- `REPORT_3WAY_BOOT_COMPARISON.md`
- `REPORT_INIT_BOOT_STOCK_VS_LIVE.md`
- `REPORT_VENDOR_BOOT_STOCK_VS_LIVE.md`
- `REPORT_DTBO_STOCK_VS_LIVE.md`
- `REPORT_VBMETA_STOCK_VS_LIVE.md`

Evidence summaries:

- `IMAGE_METADATA.txt`
- `EXTRACTED_TREE_SUMMARY.txt`
- `SEARCH_HITS_BOOT_CRITICAL.txt`
- `ACTIVE_SLOT_A_SHA256SUMS.txt`

## Raw Firmware Policy

The following are intentionally excluded from git:

- raw `.img` partition dumps
- `.bin` bootloader or firmware binaries
- `.pac` firmware packages
- extracted firmware trees
- generated comparison extraction directories
- archives and partial downloads

The reports include hashes, metadata, and analysis results without storing proprietary raw firmware content.

## Next Phase

The next main task is not another boot-chain identity check. That phase is complete.

Next phase:

- build a live Android 5.15 kernel header/module environment
- use `/proc/config.gz`
- use `/sys/kernel/kheaders.tar.xz`
- validate `CONFIG_MODVERSIONS`
- test a minimal external module
- evaluate whether USB Wi-Fi, Bluetooth, or HID modules are realistic

Plan: `NEXT_PHASE_HEADERS_MODULES_PLAN.md`.

Do not treat this repository as a universal rooting recipe. It documents one device/build state and the evidence collected from it.
