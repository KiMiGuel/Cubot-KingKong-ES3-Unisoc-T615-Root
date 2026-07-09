# Cubot KingKong ES 3 — Unisoc T615 Bootloader Unlock & Root

> **Verified Working Guide** | Unisoc T615 / UMS9230_6h10 | UFS | A/B Partitioning
> **Status:** Bootloader unlocked, FDL2 access confirmed, Magisk root confirmed
> **Build:** `CUBOT_KINGKONG_ES_3_F071_V16_20260309`
> **Host Environment:** Kali Linux / Raspberry Pi 4 recommended
> **Verified:** 2026-06-23 / 2026-06-24 by live testing

---

## ⚠️ Warnings & Disclaimer

* **This guide will wipe your data.** Flashing `misc-wipe.bin` triggers a factory reset to remove old Android encryption keys.
* **BROM access is your safety net.** As long as you can still enter BROM mode, you may be able to recover from many soft-brick situations.
* **Fastboot unlock commands are not implemented on this device.** Commands like `fastboot flashing unlock` or `fastboot oem unlock` may return `unknown cmd` or fail.
* **A/B slots:** This device uses A/B partitioning. Be aware of the active slot before flashing boot-critical partitions, or you may run into slot-mismatch boot issues.
* **Use the correct firmware files.** Using FDL files, bootloader files, or partition images from another device can brick your phone.
* **Use Linux for critical BROM/FDL work.** macOS libusb behavior can be unstable with Unisoc BROM mode.

No warranty is provided. You are responsible for any damage, data loss, boot failure, or device brick.

---

## Table of Contents

* [Device Profile](#device-profile)
* [Compatibility](#compatibility)
* [What You Need](#what-you-need)
* [Critical Discovery](#critical-discovery)
* [Step-by-Step: FDL2 Access](#step-by-step-fdl2-access)
* [Step-by-Step: Bootloader Unlock](#step-by-step-bootloader-unlock)
* [Step-by-Step: Magisk Root](#step-by-step-magisk-root)
* [Step-by-Step: NetHunter Install](#step-by-step-nethunter-install)
* [The WiFi Reality](#the-wifi-reality)
* [Troubleshooting](#troubleshooting)
* [Files Reference](#files-reference)
* [Verified Working Commands](#verified-working-commands)
* [Changelog](#changelog)
* [Credits](#credits)

---

## Device Profile

| Field                      | Observed Value                          |
| -------------------------- | --------------------------------------- |
| Device                     | Cubot KingKong ES 3                     |
| SoC                        | Unisoc T615 / UMS9230_6h10              |
| Build Number               | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Storage                    | **UFS**                                 |
| Partition Layout           | A/B slots                               |
| BROM                       | SPRD3                                   |
| Boot Key                   | Volume Down while plugging USB          |
| Magisk Target              | `boot.img` / `boot_a`                   |
| NetHunter Status           | Installable                             |
| Built-in WiFi Monitor Mode | Not supported                           |

This device was initially assumed to be EMMC during early research, but live testing confirmed it uses **UFS** storage.

---

## Compatibility

### Verified Working

* Cubot KingKong ES 3
* Unisoc T615 / UMS9230_6h10
* SPRD3 BROM
* UFS storage
* A/B partitioning

### Likely Similar

These may use similar patterns, but are not fully verified in this guide:

* Unisoc T615 devices
* Unisoc T606 / UMS9230 variants
* Unisoc T616 / UMS9230 variants
* Other SPRD3 / UMS9230 devices

### Not Covered

* MediaTek devices
* Qualcomm devices
* Samsung Exynos devices
* Unisoc T610 / T618 devices
* Unisoc T700 / T770 devices
* Devices with patched BROM
* Devices with different FDL load addresses

### Basic Compatibility Check

Power off the phone, hold **Volume Down**, and plug in USB.

On Linux:

```bash
lsusb | grep "1782:4d00"
```

If BROM mode is active, you may see:

```text
Spreadtrum Communications Inc.
1782:4d00
```

If `CHECK_BAUD` returns `SPRD3` and `CMD_CONNECT` works with `spd_dump`, the device is likely compatible with this method.

---

## What You Need

### Hardware

* Cubot KingKong ES 3 or compatible Unisoc UMS9230 device
* USB-C cable
* Linux machine
* Kali Linux recommended
* Optional USB-C OTG adapter
* Optional external USB WiFi adapter for NetHunter monitor mode / injection

### Required Files

Place all unlock files in:

```text
~/cubot_unlock/
```

Required files:

```text
custom_exec_no_verify_65015f08.bin
fdl1-dl.bin
fdl2-dl.bin
splloader.bin
misc-wipe.bin
spd_dump
```

Optional or backup files:

```text
uboot.bin
uboot_bak.bin
splloader_og.bin
boot_a.bin
boot_b.bin
vendor_boot_a.bin
dtb_a.bin
```

### Tools

* `spd_dump`
* `adb`
* `git`
* `make`
* Magisk app
* NetHunter app

---

## Get FDL Files

The safest source is your device's own stock firmware PAC file.

Extract:

```text
fdl1-dl.bin
fdl2-dl.bin
```

These may appear in firmware packages under names such as:

```text
fdl1.bin
fdl2.bin
splloader.bin
uboot.bin
```

Do not blindly use FDL files from another device. That is an excellent way to turn a phone into a very expensive rectangle.

---

## Build `spd_dump`

Clone the tool source:

```bash
git clone https://github.com/TomKing062/spreadtrum_flash.git
```

Enter the folder:

```bash
cd spreadtrum_flash
```

Build it:

```bash
make
```

Create the unlock folder:

```bash
mkdir -p ~/cubot_unlock
```

Copy `spd_dump`:

```bash
cp spd_dump ~/cubot_unlock/
```

Make it executable:

```bash
chmod +x ~/cubot_unlock/spd_dump
```

---

## Prepare Unlock Folder

Go to the unlock folder:

```bash
cd ~/cubot_unlock
```

Expected example:

```text
~/cubot_unlock/
├── custom_exec_no_verify_65015f08.bin
├── fdl1-dl.bin
├── fdl2-dl.bin
├── misc-wipe.bin
├── spd_dump
└── splloader.bin
```

---

## Critical Discovery

After repeated failed attempts with `exec_addr` and `exec_addr2`, the clean working exploit delivery method for this device was verified as:

```text
loadexec custom_exec_no_verify_65015f08.bin
```

The filename matters.

The payload must be named exactly:

```text
custom_exec_no_verify_65015f08.bin
```

The address is parsed from the filename:

```text
65015f08
```

Manual `exec_addr` and `exec_addr2` attempts failed on this device. During testing, the BROM crashed on the first FDL1 data packet when using those older methods, but stayed alive and completed the chain when using `loadexec`.

### Why `loadexec` Matters

The likely practical difference:

* `exec_addr` sends the payload as a raw binary at a manually supplied address.
* `loadexec` handles payload delivery differently.
* The filename-based address reduces manual address entry mistakes.
* On this UMS9230_6h10 target, `loadexec` is the verified working path.

Do not rename the payload unless you know exactly what you are doing.

Wrong names may break the chain.

---

## Step-by-Step: FDL2 Access

From Linux:

```bash
cd ~/cubot_unlock
```

Run the verified command:

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

### Phone Steps

1. Power off the phone completely.
2. Hold **Volume Down**.
3. While holding Volume Down, plug in USB.
4. Keep holding until `spd_dump` connects.

Expected signs of success:

```text
CHECK_BAUD bootrom
BSL_REP_VER: "SPRD3"
CMD_CONNECT bootrom
SEND fdl1-dl.bin to 0x65000800
EXEC FDL1
SEND fdl2-dl.bin to 0x9efffe00
FDL2 >
```

If you reach:

```text
FDL2 >
```

you have FDL2 access.

That is the important part.

---

## Step-by-Step: Bootloader Unlock

At the `FDL2>` prompt, flash the patched bootloader:

```text
w splloader splloader.bin
```

Trigger the factory reset:

```text
w misc misc-wipe.bin
```

Reboot:

```text
reset
```

The phone should:

1. Reboot.
2. Show recovery / wipe progress.
3. Boot to the Android setup screen.
4. Come back with the bootloader unlocked.

### Why the Wipe Is Recommended

`misc-wipe.bin` triggers a factory reset.

This removes old Android security state and avoids encryption or boot state mismatch after the bootloader unlock.

Skipping the wipe may leave the device unstable or unable to boot cleanly.

---

## Step-by-Step: Magisk Root

After bootloader unlock and initial Android setup, install Magisk.

### Method A: Direct Install

This is the easiest method if Magisk supports it after unlock.

1. Install the Magisk APK.
2. Open Magisk.
3. Choose **Install**.
4. Choose **Direct Install**.
5. Reboot.
6. Verify root.

Verify with:

```bash
adb shell su -c "id"
```

Expected result:

```text
uid=0(root)
```

### Method B: Patch Boot Image Through FDL2

If Direct Install does not work, patch and flash manually.

At the `FDL2>` prompt:

```text
r boot_a
```

Copy the dumped image to the phone:

```bash
adb push boot_a.bin /sdcard/Download/
```

Patch it in Magisk:

```text
Magisk → Install → Select and Patch a File → boot_a.bin
```

Pull the patched file back:

```bash
adb pull /sdcard/Download/magisk_patched-*.img
```

Boot into FDL2 again and flash:

```text
w boot_a magisk_patched.img
```

Reboot:

```text
reset
```

### Method C: Fastboot

If fastboot works after unlock:

```bash
adb reboot bootloader
```

Then:

```bash
fastboot flash boot_a magisk_patched.img
```

Then:

```bash
fastboot reboot
```

If fastboot commands are not implemented on your build, use FDL2 instead.

---

## Step-by-Step: NetHunter Install

After Magisk root works:

1. Download NetHunter from:

```text
https://store.nethunter.com/
```

2. Install the NetHunter app.
3. Open NetHunter.
4. Grant root access in Magisk.
5. Open Kali Chroot Manager.
6. Install Kali Chroot.
7. Choose Minimal or Full.
8. Wait for installation.

Minimal chroot generally needs around:

```text
2GB+
```

Full chroot may need around:

```text
4GB-5GB+
```

### Manual NetHunter Chroot Install

If the in-app download fails, download the rootfs manually:

```bash
wget https://kali.download/nethunter-images/current/rootfs/kalifs-arm64-minimal.tar.xz
```

Push it to the phone:

```bash
adb push kalifs-arm64-minimal.tar.xz /sdcard/
```

Then in NetHunter:

```text
Kali Chroot Manager → Install from SDCard
```

Select the `.tar.xz` file.

---

## The WiFi Reality

The built-in WiFi does **not** support monitor mode.

This is not a simple kernel config issue.

The device uses the proprietary Unisoc WiFi driver:

```text
sprd_wlan_combo
```

Observed behavior:

* `wlan0` supports normal WiFi client mode.
* `wlan0` supports AP mode.
* `wlan0` supports P2P mode.
* `wlan0` does not expose monitor mode.
* Packet injection does not work on built-in WiFi.
* This is a driver limitation.

For monitor mode and injection, use an external USB WiFi adapter.

Recommended adapters:

```text
Alfa AWUS036ACH
TP-Link TL-WN722N v1
```

Expected NetHunter path:

```text
External adapter → USB-C OTG → wlan1 or wlan2 → monitor mode
```

Example:

```bash
airmon-ng start wlan1
```

A custom kernel may still be useful for other things, but it will not magically make the proprietary `sprd_wlan_combo` driver expose monitor mode. The bottleneck is the driver stack, not just the kernel config.

---

## Troubleshooting

### `LIBUSB_ERROR_TIMEOUT`

Possible causes:

* Phone is not fully in BROM mode.
* Wrong timing.
* Bad USB cable.
* macOS libusb instability.

Fix:

* Use Linux.
* Power off completely.
* Hold Volume Down before plugging USB.
* Keep holding until connection starts.
* Try another USB cable or USB port.

---

### `LIBUSB_ERROR_IO` on FDL1 Packet

Likely cause:

```text
You are using exec_addr or exec_addr2.
```

Fix:

```text
Use loadexec only.
```

Correct command:

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

---

### `wrong command or wrong mode detected`

Likely cause:

```text
BROM crashed.
```

Fix:

1. Unplug USB.
2. Hold Power + Volume Up for around 10 seconds.
3. Wait around 10 seconds.
4. Try BROM entry again.

---

### `FDL2: incompatible partition`

This may be non-fatal.

If FDL2 still loads and you get:

```text
FDL2 >
```

continue carefully.

---

### Phone Will Not Enter BROM

Try:

```text
Volume Down
```

If that fails:

```text
Volume Up
```

If that fails:

```text
Volume Up + Volume Down
```

Also check:

* Battery charge
* USB cable
* USB port
* Whether the phone is fully powered off

---

### Magisk Direct Install Fails

Try the manual patch method.

Basic flow:

```text
Dump boot_a → patch with Magisk → flash patched image through FDL2
```

---

### NetHunter Chroot Download Fails

Check:

* Free space on `/data`
* Root access granted in Magisk
* Network connection
* NetHunter app permissions

Fallback:

```text
Use manual install from tar.xz
```

---

### No Monitor Mode on `wlan0`

Expected.

Use an external USB WiFi adapter.

---

## Files Reference

Example unlock folder:

```text
~/cubot_unlock/
├── custom_exec_no_verify_65015f08.bin
├── fdl1-dl.bin
├── fdl2-dl.bin
├── spd_dump
├── splloader.bin
├── misc-wipe.bin
├── uboot.bin
├── uboot_bak.bin
└── splloader_og.bin
```

Observed partition notes from live testing:

```text
boot_a / boot_b: 64MB each
vendor_boot_a / vendor_boot_b: 100MB each
dtb_a / dtb_b: 8MB each
init_boot_a / init_boot_b: 8MB each
super: 5600MB
userdata: ~237151MB
storage type: UFS
total partitions: 74
```

---

## Verified Working Commands

### FDL2 Access

```bash
sudo ./spd_dump --verbose 2 --wait 300 loadexec custom_exec_no_verify_65015f08.bin fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec
```

### Bootloader Unlock

```text
w splloader splloader.bin
w misc misc-wipe.bin
reset
```

### Backup Commands

```text
r boot_a
r boot_b
r dtb_a
r vendor_boot_a
```

### Flash Magisk Patched Boot

```text
w boot_a magisk_patched.img
reset
```

### Utility Commands

```text
p
reboot-recovery
reboot-fastboot
poweroff
timeout 30000
blk_size 65535
rawdata 1
```

---

## Changelog

| Date       | Change                                                |
| ---------- | ----------------------------------------------------- |
| 2026-06-23 | Initial testing and failed `exec_addr` variants       |
| 2026-06-24 | Moved to Kali Linux                                   |
| 2026-06-24 | Confirmed `loadexec` as the working method            |
| 2026-06-24 | Verified FDL2 access                                  |
| 2026-06-24 | Verified bootloader unlock                            |
| 2026-06-24 | Verified Magisk root                                  |
| 2026-06-24 | Confirmed UFS storage                                 |
| 2026-06-24 | Confirmed built-in WiFi does not support monitor mode |
| 2026-06-24 | Clean guide prepared                                  |

---

## Credits

* **@TomKing062** — Unisoc research tools and `spreadtrum_flash`
* **@topjohnwu** — Magisk
* **Kali NetHunter Project** — NetHunter platform
* **Hovatek Forum** — Unisoc bootloader and AVB research
* **Kimi AI** — Troubleshooting, reconstruction, and corrections
* **Original Tester / Maintainer:** Miguel Portugal / KiMiGuel

---

## Disclaimer

This guide is provided for research, repair, recovery, and owner-controlled device modification.

No warranty is provided.

You are responsible for any damage, data loss, boot failure, or device brick.

Use stock firmware from your own device whenever possible.

Do not flash random bootloader files across different models.

---

## Final Verified Status

```text
Device: Cubot KingKong ES 3
SoC: Unisoc T615 / UMS9230_6h10
Build: CUBOT_KINGKONG_ES_3_F071_V16_20260309
Storage: UFS
FDL2: Confirmed
Bootloader: Unlocked
Magisk root: Confirmed
NetHunter: Installable
WiFi injection: External USB adapter required
```

---

*Last Updated: 2026-06-24*
