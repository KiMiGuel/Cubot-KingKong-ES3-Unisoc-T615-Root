# Next Phase: Android 5.15 Live Headers and Module Build Environment

This plan tracks kernel-module work against the currently running stock Cubot KingKong ES3 kernel. It is not a flashing, boot image patching, or kernel replacement plan.

Target kernel:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

## Status Update

The live-header/module-build phase is no longer theoretical.

A working external-module environment was reconstructed on Kali 3 using:

```text
common-kernel commit: 4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7
toolchain: Android Clang 14.0.7 / clang-r450784e
config baseline: live /proc/config.gz
Module.symvers: exact matching symbol table
module_layout CRC: 0x0222dd63
```

Result:

- modules built on Kali 3 loaded on the Cubot stock kernel
- `ath9k_htc` stack was validated with Netgear WNA1100 / AR9271
- `wlan1` appeared
- monitor mode worked
- injection test worked in an authorized lab
- Wifite could use the adapter

See `NETHUNTER_NETGEAR_AR9271_RESULTS.md`.

## Goals

Primary goals now:

- preserve the working stock-kernel module-build method
- avoid turning the experimental AR9271 loader into a default boot-time module
- triage remaining USB Wi-Fi adapters by exact USB ID
- choose the safest next driver path before building anything

Secondary goals:

- determine whether Realtek USB adapters can be supported through a vendor driver without adding a custom `mac80211`
- determine whether ALFA/MediaTek support requires the `mt76`/`mac80211` stack
- continue research toward a coherent vendor wireless build before treating AR9271 as production-ready

Non-goals:

- do not flash images
- do not repack boot images
- do not replace the stock kernel
- do not unload stock `cfg80211` or `sprd_wlan_combo`
- do not assume a module is safe just because it builds or loads
- do not store compiled modules, firmware blobs, PAC files, or raw partition images in this repo

## Confirmed Build Environment

Important values:

| Item | Value |
|---|---|
| Device kernel | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Common kernel commit | `4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7` |
| Compiler | Android Clang 14.0.7 / `clang-r450784e` |
| `CONFIG_MODULES` | enabled |
| `CONFIG_MODVERSIONS` | enabled |
| `module_layout` CRC | `0x0222dd63` |
| External module test | successful |
| NetHunter USB Wi-Fi proof | Netgear WNA1100 / AR9271 |

This means Kali 3 and the Cubot are aligned well enough to build compatible external modules for the stock kernel when the driver dependency graph is compatible with the running kernel.

## Netgear WNA1100 / AR9271 Result

Validated module stack:

```text
mac80211.ko
ath.ko
ath9k_hw.ko
ath9k_common.ko
ath9k_htc.ko
```

Do not load a custom `cfg80211.ko`. The Cubot already loads stock `cfg80211`, and the internal Unisoc Wi-Fi driver depends on it.

Validated capabilities:

```text
wlan1 creation
monitor mode
packet injection test
Wifite usage
```

Important limitation:

The current automatic/Magisk loader design is experimental. It should remain disabled until redesigned and retested. The safe claim is that the adapter was made to work manually through externally built stock-kernel modules.

## Adapter Triage Plan

The next work should test other adapters before continuing to obsess over AR9271. A deeply human move, realizing the first rabbit hole was only one rabbit. Annoying, but useful.

Priority order:

1. TP-Link Realtek adapter
2. ALFA adapter
3. Netgear WNA1100 / AR9271 refinement

First collect exact USB IDs:

```sh
lsusb -nn | grep -Ei 'Realtek|MediaTek|Ralink|Atheros|Qualcomm|TP-Link|Alfa|Netgear|0bda|148f|0cf3|0846'
```

For each adapter, record:

- vendor ID
- product ID
- marketing model if known
- chipset
- expected Linux driver
- required module stack
- whether the path depends on custom `mac80211`

## Driver Risk Classification

| Driver path | Expected risk | Notes |
|---|---|---|
| Realtek `88XXau` vendor driver | Green/Yellow | Potentially best first candidate if it can consume stock `cfg80211` without custom `mac80211` |
| Realtek `rtw88_usb` | Yellow/Red | Cleaner upstream path, but likely requires `mac80211` |
| MediaTek `mt76x0u` | Yellow/Red | Likely requires `mt76` plus `mac80211` |
| AR9271 `ath9k_htc` | Red for production | Proven functional, but current hybrid stack caused input instability |

## Cubot Stock Kernel Capability Checks

Collect read-only evidence before any new driver build:

```sh
adb shell su -c 'uname -a'
adb shell su -c 'uname -r'
adb shell su -c 'zcat /proc/config.gz' > live_kernel.config
adb shell su -c 'cat /proc/version'
adb shell su -c 'cat /proc/modules' > live_proc_modules.txt
adb shell su -c 'find /vendor_dlkm/lib/modules -maxdepth 1 -type f -name "*.ko" | sort'
adb shell su -c 'cat /vendor_dlkm/lib/modules/modules.dep 2>/dev/null || true'
adb shell su -c 'cat /vendor_dlkm/lib/modules/modules.alias 2>/dev/null || true'
```

Check configuration values:

```sh
grep -E '^(CONFIG_CFG80211|CONFIG_MAC80211|CONFIG_WLAN_VENDOR_REALTEK|CONFIG_RTW88|CONFIG_RTW88_USB|CONFIG_MT76|CONFIG_MT76_USB|CONFIG_MT76x0U|CONFIG_MT76x2U|CONFIG_ATH9K_HTC|CONFIG_USB|CONFIG_HID|CONFIG_BT)=' live_kernel.config
```

Search stock modules:

```sh
adb shell su -c 'find /vendor_dlkm/lib/modules -maxdepth 1 -type f | grep -Ei "rtl|rtw|881|882|88xx|mt76|mt7610|mt7601|ath|cfg80211|mac80211" || true'
```

## Local Source Audit

Search local source/workspace roots only:

```text
/home/kali/Kernels
/home/kali/Kernels/cubot_gki_13530115
/home/kali/Kernels/hmd_arc_kernel
```

Candidate names:

```text
rtl8812au
8812au
88XXau
rtl88xxau
rtw88
mt76
mt76x0u
mt7610u
ath9k_htc
```

For each source candidate, record:

- path
- git remote
- branch/commit
- Android support present or absent
- Makefile platform options
- required kernel subsystems
- whether it requires `mac80211`
- whether it can build as an external module against stock `cfg80211`
- known Android 5.15 build problems visible in the source

## Stop Conditions

Stop immediately if:

- a driver path requires replacing stock `cfg80211` without a coherent vendor build plan
- a module depends on unexported symbols that require unsafe compatibility patches
- a module load affects touchscreen/input, stock Wi-Fi, USB, or system stability
- the adapter requires firmware that cannot be legally or safely distributed here
- the work would require flashing or replacing the active kernel

## Deliverables

Suggested future report outputs:

```text
NETHUNTER_NETGEAR_AR9271_RESULTS.md
NETHUNTER_USB_ADAPTER_TRIAGE.md
REALTEK_RTL8811AU_BUILD_NOTES.md
ALFA_MT7610U_BUILD_NOTES.md
COHERENT_WIRELESS_STACK_PLAN.md
```

## Current Recommendation

Leave the AR9271 Magisk module disabled.

Next safest work:

1. identify TP-Link and ALFA adapters by USB ID
2. audit Realtek `88XXau` first
3. prefer a driver path that does not require custom `mac80211`
4. only build after the dependency graph is understood
5. test one adapter and one module stack at a time
