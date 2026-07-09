# PLAN.md - Cubot KingKong ES3 Root Research Plan

## Objective

Document the active boot chain and root state for the Cubot KingKong ES3, then move into a controlled live Android 5.15 kernel header/module build environment.

This repo is documentation and analysis focused. Raw firmware images and extracted proprietary trees remain outside git.

## Confirmed Device Context

| Field | Value |
|---|---|
| Device | Cubot KingKong ES3 |
| SoC | Unisoc T615 / ums9230-family |
| Android build | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Kernel | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Active slot | `_a` |
| Storage | UFS |
| Block path family | `/dev/block/sda*` |

Active slot map:

| Partition | Block device |
|---|---|
| `boot_a` | `/dev/block/sda36` |
| `vendor_boot_a` | `/dev/block/sda38` |
| `init_boot_a` | `/dev/block/sda40` |
| `dtbo_a` | `/dev/block/sda44` |
| `vbmeta_a` | `/dev/block/sda49` |

Do not treat this device as eMMC. The command line confirms UFS with `sprdoot.flash=ufs`.

## Completed Phases

| Phase | Status | Output |
|---|---:|---|
| Active slot dump | complete | `ACTIVE_SLOT_A_SHA256SUMS.txt`, `../slot_a_ordered/` |
| Stock PAC extraction | complete | `/home/kali/pacextractor/pac_out/` |
| Initial active-slot metadata | complete | `IMAGE_METADATA.txt` |
| Active-slot extraction summary | complete | `EXTRACTED_TREE_SUMMARY.txt` |
| Boot-critical search | complete | `SEARCH_HITS_BOOT_CRITICAL.txt` |
| Boot stock/live/custom comparison | complete | `REPORT_3WAY_BOOT_COMPARISON.md` |
| init_boot stock/live comparison | complete | `REPORT_INIT_BOOT_STOCK_VS_LIVE.md` |
| vendor_boot stock/live comparison | complete | `REPORT_VENDOR_BOOT_STOCK_VS_LIVE.md` |
| dtbo stock/live comparison | complete | `REPORT_DTBO_STOCK_VS_LIVE.md` |
| vbmeta stock/live comparison | complete | `REPORT_VBMETA_STOCK_VS_LIVE.md` |
| LK warning origin analysis | complete | summarized in `BOOT_CHAIN_FINDINGS.md` |
| Final boot-chain summary | complete | `BOOT_CHAIN_FINDINGS.md` |
| Root guide cleanup | complete | `README.md`, `ROOTING_GUIDE.md`, `REPORT_INDEX.md` |

## Current Findings

| Component | Finding |
|---|---|
| `boot_a` | stock PAC match; stock kernel active |
| `init_boot_a` | Magisk-patched; root source |
| `vendor_boot_a` | stock PAC match |
| `dtbo_a` | stock PAC match |
| `vbmeta_a` | one-byte modified flags field; stock descriptors |
| LK | unlocked/orange behavior; warning originates in LK-stage bootloader |

The device is rooted, but custom kernel work remains unsolved.

## Next Phase - Android 5.15 Live kheaders and Modules

Plan: `NEXT_PHASE_HEADERS_MODULES_PLAN.md`.

Goal:

- Build and validate modules against the currently running stock kernel.

Inputs expected from the phone:

```text
/proc/config.gz
/sys/kernel/kheaders.tar.xz
uname -r
/proc/modules
/proc/kallsyms if readable
Module.symvers if available
```

Main risks:

- `CONFIG_MODVERSIONS=y`
- missing or mismatched `Module.symvers`
- `module_layout` CRC mismatch
- `invalid module format`
- unknown symbols
- mismatched vermagic
- missing firmware for USB devices

Stop before any work that requires flashing, repacking boot images, or replacing the active kernel.

## Standing Rules

- Do not modify original image files.
- Do not flash anything during analysis phases.
- Do not repack or generate boot images unless explicitly requested in a future phase.
- Do not copy raw `.img`, `.bin`, or `.pac` files into git.
- Keep extracted artifact folders ignored unless there is a deliberate reason to track a small text artifact.
- Clearly separate proven facts from hypotheses.
