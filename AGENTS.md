# AGENTS.md - Cubot KingKong ES3 Active Slot A Analysis

## Mission

Analyze the currently booting Cubot KingKong ES3 active slot A image set.

Device:
- Cubot KingKong ES3
- SoC: Unisoc T615 / ums9230-style platform
- Android 13
- Kernel release: 5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
- Active slot: _a
- Storage: UFS
- Boot devices use /dev/block/sda*

Confirmed active slot map:
- boot_a        -> /dev/block/sda36
- vendor_boot_a -> /dev/block/sda38
- init_boot_a   -> /dev/block/sda40
- dtbo_a         -> /dev/block/sda44
- vbmeta_a       -> /dev/block/sda49

Input images:
../slot_a_ordered/00_vbmeta_a_live.img
../slot_a_ordered/01_dtbo_a_live.img
../slot_a_ordered/02_boot_a_live.img
../slot_a_ordered/03_vendor_boot_a_live.img
../slot_a_ordered/04_init_boot_a_live.img

Hash manifest:
../ACTIVE_SLOT_A_SHA256SUMS.txt

## Hard Rules

1. Do not modify original image files.
2. Do not flash anything.
3. Do not generate patched boot images unless explicitly asked.
4. Do not assume this is eMMC. The command line confirms UFS:
   sprdoot.flash=ufs
5. Do not chase generic Linux root=/dev/mmcblk0p2 advice.
6. Do not hex-patch kernel panic call trace addresses.
7. Treat panic addresses as runtime kernel addresses only.
8. Preserve all extracted files under this workspace.
9. Write findings to markdown reports.
10. If a tool is missing, report the missing tool and provide the exact apt/pip/git install command, but do not derail the analysis.

## Known Panic Context

Panic screen showed:

Reset mode: kernel_crash
exception_file_info: not-bugon
exception_panic_reason:
Attempted to kill init! exitcode=0x00007f00

Also shown:

init_mmc_fat failed, Please check SD Card!!!!
Try to do dump to PC now !!
Dump to PC start..

Interpretation:
- PID 1 init exited.
- Kernel panic is likely secondary.
- init_mmc_fat may be crashdump/sysdump related and should not be treated as proof that Android boot storage is eMMC.
- The active Android boot path is UFS / sda*.

## Known Suspect Area

Previous suspected root cause:
- UFS storage path must come up early.
- CONFIG_SCSI_UFS_SPRD may be built-in.
- SC2730 regulator support may be missing, late, or mismatched.
- UFS DT node may depend on SC2730 regulators:
  - vcc
  - vddgen0
  - avdd12
  - avdd18
  - vddcore
  - vddmodem
- Module version mismatch may block vendor modules:
  - CONFIG_MODVERSIONS=y
  - module_layout CRC mismatch
  - printk_cpuid.ko failure
  - sc2730-regulator.ko / ufs_sprd.ko may fail to load

## Analysis Goals

1. Verify image types and boot image versions.
2. Extract boot_a, init_boot_a, vendor_boot_a, dtbo_a, and vbmeta_a where possible.
3. Identify kernel command line, ramdisk contents, fstab files, vendor ramdisk modules, and DTBO contents.
4. Search extracted files for:
   - fstab
   - first_stage_mount
   - ufs
   - sprd
   - sc2730
   - regulator
   - sda
   - by-name
   - module_layout
   - vermagic
5. Determine whether active slot A looks stock, rooted, patched, or mismatched.
6. Produce a report:
   - REPORT_ACTIVE_SLOT_A_ANALYSIS.md
7. Produce a tool/install report if needed:
   - REPORT_MISSING_TOOLS.md

## Deliverables

- REPORT_ACTIVE_SLOT_A_ANALYSIS.md
- EXTRACTED_TREE_SUMMARY.txt
- SEARCH_HITS_BOOT_CRITICAL.txt
- IMAGE_METADATA.txt
- Any unpacked files inside ./extracted/
