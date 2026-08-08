**En Español haz click [aquí](./README_ES.md)**

# 🔓 Cubot KingKong ES 3 — Unlock & Root Guide

**Unisoc T615 (UMS9230_6h10) · UFS · A/B slots · build `CUBOT_KINGKONG_ES_3_F071_V16_20260309`**

Follows the same 5-step flow as the classic `unlock_autopatch_9230` script — plus tool prep (Step 0) and Magisk root (Step 6).

> ⚠️ **Read this first**
> - 🟡 Difficulty: medium/hard. If you don't know what you're doing, stop here.
> - 💥 **This wipes the device** — back up first.
> - 🔌 **USB 2.0 port + cable only** — USB 3.0 causes connection failures on this device.
> - 🐧 Use **Linux** (Kali). Not macOS, not Windows.
> - 🎯 Verified only on the build above — re-verify before using on any other build.
> - 🧯 Bricked mid-way? BROM mode always answers — you can always recover.

---

## 🧰 Step 0 — Set up the tools

Download the `cubot_es3_unlock_package.zip` package from [Releases](https://github.com/KiMiGuel/Cubot-KingKong-ES3-Unisoc-T615-Root/releases), unzip, and build:

```
cd spl_unlock_linux
make
gcc gen_spl-unlock.c -o gen_spl-unlock
gcc chsize.c -o chsize
```

Verify you built the right thing — both checks must pass:

```
./spd_dump --help          # must NOT list a "baudrate" command
grep reopen_port common.c  # must find the reconnect fix
```

One file is NOT in the package: `init_boot.img` — extract it from **your own** stock PAC (build above). Stay inside the package folder the whole time — dumps land in the current directory.

---

## 💾 Step 1 — Back up & patch your loaders

Enter **BROM**: power off → hold **Volume Down** → plug USB → keep holding. Confirm:

```
lsusb | grep -i '1782:4d00'
```

Dump `splloader` + `uboot`:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset
```

Wait for the reset. `find port failed` → close and re-run — do **not** continue past that error.

Generate the unlock payload, resize + stash the backups:

```
gen_spl-unlock splloader.bin
chsize uboot.bin
mv splloader.bin u-boot-spl-16k-sign.bin
mv uboot.bin uboot_bak.bin
```

---

## ⚡ Step 2 — Write the working FDL2

Re-enter BROM (Step 1), then:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
```

Wait ~10 seconds after the reset.

---

## 🔓 Step 3 — Run the unlock

Re-enter BROM, then:

```
spd_dump exec_addr 0x65015f08 fdl spl-unlock.bin 0x65000800
```

What you'll see, and why it's all good:
- 🔄 `CHECK_BAUD FAIL` — expected. The patched build reconnects by itself.
- ✅ `CHECK_BAUD FDL1` → `BSL_REP_VER: "SPRD3"` → `CMD_CONNECT FDL1` — **this IS the success signal.** The unlock has already applied.
- ⏱️ `timeout reached` after that — **this is what you WANT.** Not an error. Do **not** re-run; move to Step 4. (Only if you never see `CMD_CONNECT FDL1`, run it once more.)

---

## ✅ Step 4 — Verify the unlock

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec verbose 2 read_part miscdata 8192 64 m.bin reset
```

Check `m.bin`: 64 zero bytes = still locked (repeat Step 3) · 32-byte string + two 16-byte hashes = **unlocked** ✅. A trailing timeout here is normal.

---

## 🧹 Step 5 — Restore & wipe

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

Let the phone factory-reset and boot. 🟠 **Orange/unlocked warning screen = success.**

---

## 🪄 Step 6 — Root with Magisk

Boot Android → Settings → About Phone → tap Build Number ×7 → Developer Options → **USB debugging**. Then:

```
adb push init_boot.img /sdcard/Download/
```

In the Magisk app: **Install → Select and Patch a File → `init_boot.img`**, pull the patched image back, and flash it — in **fastbootd**, not the plain bootloader:

```
adb pull /sdcard/Download/magisk_patched-*.img .
adb reboot fastboot
fastboot flash init_boot magisk_patched-XXXXX.img
fastboot reboot
```

Open Magisk → root active. 🎉 **Done — unlocked and rooted.**

---

## 🩹 Troubleshooting

| Symptom | Fix |
|---|---|
| `LIBUSB_ERROR_BUSY` | `sudo systemctl stop ModemManager`, replug |
| `LIBUSB_ERROR_NO_DEVICE` | Start `spd_dump` first, **then** plug the phone |
| `find port failed` | Close and re-run the same command |
| Two `CHECK_BAUD FAIL` → `device removed, exiting...` | Your `spd_dump` build is missing the reconnect fix — rebuild from the package source (`common.c` must contain `reopen_port`) |
| Boot loop / soft brick at any point | BROM (Step 1) always answers — reflash stock firmware, redo the guide |
| ☠️ In `spd_dump` | **Never** run `repartition`, `erase_all`, or `ufs repart/rawerase/setattr` — they truncate the GPT |

---

## 📚 Want the "why"? Full research lives here

- [`ROOTING_GUIDE.md`](ROOTING_GUIDE.md) — long-form guide with the evidence behind each step
- [`BOOT_CHAIN_FINDINGS.md`](BOOT_CHAIN_FINDINGS.md) — boot-chain / AVB analysis
- [`INVESTIGACION_COMPLETA_ES.md`](INVESTIGACION_COMPLETA_ES.md) — investigación completa en español
- [`NETHUNTER_NETGEAR_AR9271_RESULTS.md`](NETHUNTER_NETGEAR_AR9271_RESULTS.md) — NetHunter + external Wi-Fi monitor mode on the stock kernel
- [`REPORT_INDEX.md`](REPORT_INDEX.md) — stock-vs-live partition comparison series
- [`NEXT_PHASE_HEADERS_MODULES_PLAN.md`](NEXT_PHASE_HEADERS_MODULES_PLAN.md) — what comes next

This repo stores **docs and hashes only** — no firmware images, PACs, partition dumps, or `.ko` files. It documents one device/build state; don't treat it as a universal rooting recipe.

🙏 Built on [TomKing062's CVE-2022-38694 toolkit](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader) (`spd_dump`/`spreadtrum_flash`, `gen_spl-unlock`, `chsize`) and [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu. Thank You @TomKing062 @topjohnwu
