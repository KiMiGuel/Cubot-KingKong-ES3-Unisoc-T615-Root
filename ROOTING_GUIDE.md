# Cubot KingKong ES3 Rooting Guide

This guide documents the root process model confirmed by the boot-chain analysis in this repository. It is specific to the analyzed Cubot KingKong ES3 build and should not be blindly reused on other firmware.

Target device/build:

```text
Cubot KingKong ES3
Unisoc T615 / ums9230-family
CUBOT_KINGKONG_ES_3_F071_V16_20260309
Android 13
Kernel: 5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
Storage: UFS
```

## Critical Update

Earlier notes treated `boot_a` as the Magisk target. The final evidence disproves that for the rooted state analyzed here.

For this Android 13/GKI-style build:

- `boot_a` is stock and contains the stock kernel.
- Magisk root is carried by patched `init_boot_a`.
- `vendor_boot_a` and `dtbo_a` remain stock.
- `vbmeta_a` has stock descriptors but a one-byte modified signed flags field.
- LK reports unlocked/orange state and skips verification.

Do not use root success as proof that a custom kernel works.

## Warnings

This process can wipe data, break boot, or brick the device if done incorrectly.

Important rules:

- Back up boot-critical partitions before writing anything.
- Use firmware files from the exact device/build whenever possible.
- Do not use random FDL, SPL, LK, or boot images from another phone.
- Do not treat this UFS device as eMMC.
- Do not flash `boot_a` for Magisk root on this build unless later evidence proves your target differs.
- Do not flash custom kernels as part of basic root.
- Keep known-good stock PAC files and partition dumps outside git.

This repository does not store raw firmware images.

## Required Inputs

From the stock PAC extraction:

```text
fdl1-dl.bin
fdl2-dl.bin
init_boot.img
vbmeta-sign.img
```

From the unlock/root working folder used during live testing:

```text
custom_exec_no_verify_65015f08.bin
splloader.bin
misc-wipe.bin
spd_dump
```

Optional but strongly recommended backups:

```text
boot_a.img
vendor_boot_a.img
dtbo_a.img
init_boot_a.img
vbmeta_a.img
splloader_original.bin
```

## Build or Prepare spd_dump

The previous working flow used `spd_dump` from the Spreadtrum/Unisoc flashing tooling.

Example build flow:

```sh
git clone https://github.com/TomKing062/spreadtrum_flash.git
cd spreadtrum_flash
make
mkdir -p ~/cubot_unlock
cp spd_dump ~/cubot_unlock/
chmod +x ~/cubot_unlock/spd_dump
```

Use Linux for the BROM/FDL steps. macOS USB/libusb behavior can be unreliable for this path.

## Enter BROM Mode

1. Power off the phone.
2. Hold Volume Down.
3. Plug in USB while holding Volume Down.
4. Confirm Spreadtrum/Unisoc BROM appears on the host.

Example check:

```sh
lsusb | grep -i '1782:4d00\|spreadtrum\|unisoc'
```

## FDL2 Access

The working access path used `loadexec`, not `exec_addr` or `exec_addr2`.

From the unlock folder:

```sh
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

Expected result:

```text
FDL2 >
```

Once at `FDL2>`, the tool has low-level partition read/write access. Treat every write as destructive.

## Bootloader Unlock Flow

The earlier working unlock flow wrote the patched bootloader component and wipe trigger:

```text
w splloader splloader.bin
w misc misc-wipe.bin
```

`misc-wipe.bin` triggers the required wipe/factory reset behavior for the unlock flow.

After reboot:

- Android should factory reset.
- LK should report unlocked/orange behavior.
- The bootloader warning is expected.

Relevant LK strings found in the stock bootloader files:

```text
INFO: LOCK FLAG IS : UNLOCK!!!
WARNNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY!!!
bootloader has been unlocked
androidboot.verifiedbootstate=orange
```

The warning originates from LK-stage bootloader files, not from Android, Magisk, `init_boot`, or the Linux kernel.

## Magisk Root Flow for This Build

For this build, patch `init_boot.img`, not `boot.img`.

High-level flow:

1. Extract `init_boot.img` from the matching stock PAC.
2. Transfer `init_boot.img` to the phone.
3. Patch it with the Magisk app.
4. Transfer the Magisk-patched image back to the host.
5. Write the patched image to the active `init_boot` slot.
6. Boot Android and verify Magisk root.

Example transfer to phone:

```sh
adb push init_boot.img /sdcard/Download/
```

In Magisk:

```text
Install
Select and Patch a File
Choose init_boot.img
```

Copy the patched file back:

```sh
adb pull /sdcard/Download/magisk_patched-*.img .
```

After returning to `FDL2>`, write the patched init_boot to the active slot A partition:

```text
w init_boot_a magisk_patched-XXXXX.img
```

Use the actual patched file name produced by Magisk.

Do not write the Magisk-patched file to `boot_a` for this build. The verified rooted state has stock `boot_a` and patched `init_boot_a`.

## vbmeta State

The live rooted device has a modified `vbmeta_a`, but not a rebuilt descriptor map.

Observed result:

| Field | Stock | Live rooted |
|---|---:|---:|
| AVB flags | `0` | `33554432` |

The live `vbmeta_a` differs from stock by exactly one byte in the signed AVB header flags field. All descriptors, keys, rollback metadata, release string, and build properties match stock.

`avbtool` verifies the stock vbmeta struct signature but reports signature failure on the live modified `vbmeta_a`. The rooted phone can still boot because LK reports unlocked/orange state and skips verification.

## What Should Remain Stock

The confirmed rooted state keeps these stock:

```text
boot_a
vendor_boot_a
dtbo_a
```

Why this matters:

- `boot_a` contains the stock running kernel.
- `vendor_boot_a` contains boot-critical DTB, bootconfig, fstab, vendor ramdisk, and modules.
- `dtbo_a` contains stock DTBO overlays for UFS/regulator/device-tree behavior.

Changing those components is custom kernel or platform bring-up work, not basic Magisk root.

## Verification After Root

After dumping active slot A, expected evidence for this rooted state is:

| Check | Expected result |
|---|---|
| `boot_a` vs stock `boot-gki.img` | exact match |
| `init_boot_a` vs stock `init_boot.img` | differs |
| live `init_boot_a` ramdisk | Magisk overlay and backup files present |
| `vendor_boot_a` vs stock `vendor_boot.img` | exact match |
| `dtbo_a` vs stock `dtbo.img` | exact match |
| `vbmeta_a` descriptors | match stock |
| `vbmeta_a` flags | differ from stock |
| bootloader state | unlocked/orange |

Magisk evidence expected in live `init_boot_a`:

```text
.backup/.magisk
.backup/init.xz
.backup/.rmlist
overlay.d/sbin/magisk.xz
overlay.d/sbin/init-ld.xz
overlay.d/sbin/stub.xz
```

Root-state map:

```text
LK unlocked/orange
  |
  v
modified vbmeta_a flags byte, stock descriptors
  |
  v
stock boot_a kernel
  |
  v
stock vendor_boot_a and dtbo_a
  |
  v
patched init_boot_a
  |
  v
Magisk root
```

## NetHunter Notes

NetHunter Lite runs on this rooted stock-kernel state, but NetHunter userspace does not automatically provide Android kernel drivers.

The major update is that the stock Cubot kernel can load externally built modules when they are built against the correct live ABI.

Confirmed build target:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

Confirmed module-build requirements:

- exact common-kernel source commit: `4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7`
- live `/proc/config.gz`
- matching `Module.symvers`
- Android Clang 14.0.7 / `clang-r450784e`
- matching `module_layout` CRC: `0x0222dd63`

### Netgear WNA1100 / AR9271 result

The Netgear WNA1100 was validated on the rooted stock Cubot kernel using the `ath9k_htc` stack.

Validated:

- AR9271 USB adapter detected
- firmware loaded
- `wlan1` appeared
- monitor mode worked
- injection test worked in an authorized lab
- Wifite could use the adapter

Module stack used:

```text
mac80211.ko
ath.ko
ath9k_hw.ko
ath9k_common.ko
ath9k_htc.ko
```

Firmware used:

```text
htc_9271.fw
ath9k_htc/htc_9271-1.4.0.fw
```

Safety warning: the current automatic/Magisk loader is experimental and should remain disabled until redesigned. The verified milestone is manual external-module functionality, not a production-safe install package.

See `NETHUNTER_NETGEAR_AR9271_RESULTS.md` for details.

### Remaining adapter work

Do not assume every NetHunter-supported chipset now works on the Cubot.

Next adapters to triage:

- TP-Link Realtek adapter, likely `RTL8811AU` / `RTL8812AU` family
- ALFA adapter, likely MediaTek `MT7610U` / `mt76x0u` family

Each must be identified by USB ID before choosing a driver path.

## Custom Kernel Warning

The previous custom kernel boot failure is likely unrelated to Magisk root itself.

The most plausible risk chain is:

```text
custom boot kernel
  |
  v
stock vendor_boot modules, fstab, DTB, and DTBO overlays
  |
  v
module ABI or UFS/regulator mismatch
  |
  v
first-stage mount failure
  |
  v
Attempted to kill init! exitcode=0x00007f00
```

Specific risk areas:

- `CONFIG_MODVERSIONS`
- `module_layout` CRC mismatch
- `ufs_sprd.ko`
- `sc2730-regulator.ko`
- `sprd-pmic-spi.ko`
- UFS device path `/dev/block/sda*`
- SC2730 regulator rails used by the UFS DT node

Custom kernel work should proceed through `NEXT_PHASE_HEADERS_MODULES_PLAN.md`, not by assuming the current root setup validates a custom boot image.

## Files Not Stored Here

This repo intentionally does not include:

- PAC firmware packages
- raw `.img` partition dumps
- `.bin` bootloader binaries
- Magisk-patched images
- extracted proprietary firmware trees
- compiled kernel modules
- firmware blobs

Use your own stock firmware and verified dumps.
