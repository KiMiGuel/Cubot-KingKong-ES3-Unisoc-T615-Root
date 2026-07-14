# NetHunter Netgear WNA1100 / AR9271 Results

This report documents the first confirmed USB Wi-Fi milestone on the rooted Cubot KingKong ES3 stock kernel.

It does not store compiled modules, firmware blobs, PAC firmware, or raw partition images.

## Summary

Netgear WNA1100 / AR9271 support was achieved on the rooted stock Cubot kernel through externally built modules that match the live kernel ABI.

Confirmed target kernel:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

Validated result:

- AR9271 USB adapter detected
- firmware loaded
- `wlan1` appeared
- monitor mode worked
- packet injection test worked in an authorized lab
- Wifite could use the adapter

This is a stock-kernel external-module result, not a full custom-kernel boot result.

## Device and Environment

| Item | Value |
|---|---|
| Device | Cubot KingKong ES3 |
| SoC | Unisoc T615 / UMS9230 family |
| Android build | `CUBOT_KINGKONG_ES_3_F071_V16_20260309` |
| Kernel | `5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115` |
| Root | Magisk |
| NetHunter | NetHunter Lite / Kali chroot |
| Adapter | Netgear WNA1100 |
| Chipset | Qualcomm Atheros AR9271 |
| USB ID | `0846:9030` |
| Driver | `ath9k_htc` |

## Matching Kernel Module Environment

The working module environment used:

```text
common-kernel commit: 4ea0fcb5d1308f2f5a5dec0a3a5c8f1b261e00c7
toolchain: Android Clang 14.0.7 / clang-r450784e
config baseline: live /proc/config.gz
Module.symvers: exact matching symbol table
module_layout CRC: 0x0222dd63
```

Relevant stock-kernel facts:

```text
CONFIG_MODULES=y
CONFIG_MODVERSIONS=y
```

The exact kernel release string was preserved so the external modules matched the live kernel:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

## Built Module Stack

The following external modules were built and staged:

```text
mac80211.ko
ath.ko
ath9k_hw.ko
ath9k_common.ko
ath9k_htc.ko
```

A custom `cfg80211.ko` was also built during research, but it must not be loaded over the stock Cubot `cfg80211`. The stock internal Unisoc Wi-Fi stack depends on the stock `cfg80211` and `sprd_wlan_combo` runtime combination.

Dependency order:

```text
stock cfg80211
  ├── mac80211
  │     └── ath9k_htc
  └── ath
        └── ath9k_hw
              └── ath9k_common
                    └── ath9k_htc
```

Practical load order used during successful manual testing:

```text
mac80211
ath
ath9k_hw
ath9k_common
ath9k_htc
```

## Firmware

The AR9271 firmware was built and staged separately.

Firmware filenames used:

```text
htc_9271.fw
ath9k_htc/htc_9271-1.4.0.fw
```

During testing, firmware was staged outside this repository. Do not commit firmware blobs here unless license review is completed.

## Validated Runtime Behavior

After the module stack and firmware were present, the Netgear adapter produced:

```text
wlan1
```

Confirmed operations:

```text
ip link set wlan1 down
iw dev wlan1 set type monitor
ip link set wlan1 up
iw dev wlan1 info
```

Observed monitor-mode state:

```text
Interface wlan1
        type monitor
        channel 1 (2412 MHz), width: 20 MHz, center1: 2412 MHz
        txpower 20.00 dBm
```

A packet injection test succeeded in an authorized lab against the user's own router.

Wifite was also able to use the adapter.

## Important Safety Notes

The current automatic/Magisk-loader packaging is not considered production-safe.

Observed risk:

- the manual AR9271 stack worked
- later loader/module experiments were associated with touchscreen/input instability
- reboot restored stock touchscreen and Wi-Fi behavior

Therefore:

- keep the experimental AR9271 Magisk module disabled unless actively debugging it
- do not auto-load this stack at boot
- do not unload stock `cfg80211`
- do not unload `sprd_wlan_combo`
- do not replace stock vendor wireless modules without a coherent vendor wireless build plan

The safe claim is:

```text
AR9271 can be made to work manually on the rooted stock Cubot kernel with matching external modules.
```

The unsafe claim would be:

```text
AR9271 support is production-ready as a Magisk module.
```

That is not proven.

## Architecture Caveat

The successful AR9271 test used a hybrid wireless runtime:

```text
stock Cubot cfg80211
stock Cubot sprd_wlan_combo
custom mac80211
custom ath/ath9k stack
```

This passed symbol/version checks and worked for monitor mode and injection, but runtime compatibility of mixed vendor and custom wireless subsystems is not fully validated.

The better long-term architecture is a coherent wireless build in which these components are built from one compatible source/configuration environment:

```text
cfg80211
mac80211
ath
ath9k_hw
ath9k_common
ath9k_htc
sprd_wlan_combo
unisoc_wcn_bsp
required SIPC/WCN dependencies
```

`=y` versus `=m` is not the real fix. The real issue is coherent subsystem compatibility.

## What This Achievement Proves

This work proves:

- the Cubot stock kernel can load externally built modules
- Kali 3 can build modules against the live Cubot kernel ABI
- `CONFIG_MODVERSIONS` and `module_layout` matching were solved
- NetHunter Lite can use an external USB Wi-Fi adapter when the Android kernel driver stack is supplied
- AR9271 monitor mode and injection are possible on this device/build

This work does not prove:

- a custom kernel boots
- every NetHunter-supported adapter works
- Realtek or MediaTek adapters are solved
- automatic boot-time loading is safe
- the AR9271 stack is production-ready

## Next Adapter Work

The next adapters should be identified by USB ID before choosing drivers.

Suggested priority:

1. TP-Link Realtek adapter, likely `RTL8811AU` / `RTL8812AU`
2. ALFA adapter, likely MediaTek `MT7610U` / `mt76x0u`
3. AR9271 loader hardening only after the above triage is complete

First command for adapter inventory:

```sh
lsusb -nn | grep -Ei 'Realtek|MediaTek|Ralink|Atheros|Qualcomm|TP-Link|Alfa|Netgear|0bda|148f|0cf3|0846'
```

## Repository Policy

This repository should store:

- documentation
- hashes
- command logs when sanitized
- build notes
- dependency reports
- safety findings

This repository should not store:

- compiled `.ko` files
- firmware blobs
- PAC firmware packages
- raw partition dumps
- Magisk-patched boot images
- proprietary extracted vendor trees
