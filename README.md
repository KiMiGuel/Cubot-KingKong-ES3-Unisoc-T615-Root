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
> - ☢️ `prodnv` / `nvitem` / `l_fixnv` hold factory calibration data — never read/write/erase them, at any step.

---

## 🗺️ The cast of characters — read this once and every command below makes sense

Every command in this guide does one of two things: talk to **BROM**, or (once the helpers are loaded) talk to **FDL2**. Here's who's who:

- **BROM (Boot ROM)** — code burned into the Unisoc chip at the factory. It can never be erased or bricked, which is why it's your panic button. With the phone powered off, holding a volume key while plugging in USB wakes it up; the phone then shows up in `lsusb` as `1782:4d00`. **BROM does nothing on its own** — it just waits for a PC to send it code.
- **The exploit (`exec_addr 0x65015f08`)** — BROM normally refuses to run anything that isn't factory-signed. CVE-2022-38694 is a bug in this chip's BROM that lets us smuggle our own code past that check. You'll see `exec_addr 0x65015f08` at the front of almost every command — that's the crowbar prying BROM open. It works by loading `custom_exec_no_verify_65015f08.bin`, which `spd_dump` picks up automatically from the current folder (one more reason to stay inside the package folder).
- **FDL1 / FDL2** — the two helper programs we send through the pried-open BROM. **FDL1** wakes up the phone's RAM; **FDL2** is the real worker that can read, write, and erase any flash partition. The repeating pattern `fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec` simply means: *"load FDL1, load FDL2, start them."*
- **spd_dump verbs** — once FDL2 is running: `r <partition>` = copy a partition from the phone to a file · `w <partition> <file>` = write a file to a partition · `e <partition>` = erase · `read_part` = read raw bytes into a file · `reset` = reboot the phone.
- **splloader** — the first stage of the phone's *normal* boot chain, stored in its own flash partition. **It contains the bootloader lock check** — that's why it's the target. You never "extract" the splloader from any file: in Step 1 you copy it off the phone (`r splloader`) and patch that copy.
- **uboot** — the regular bootloader (the thing that gives you fastboot). In Step 2 it's temporarily replaced with a cooperative copy (`fdl2-cboot.bin`) so the unlock can run; Step 5 puts your original back.
- **miscdata** — a small hidden partition that holds the unlock flag. "Unlocking" = flipping that flag. Step 4 reads it back as proof.
- **The boot chain, in order:** `BROM → splloader → uboot → Android`. The unlock (Steps 1–5) neuters the lock check in **splloader**. The root (Step 6) is a separate trick that patches **`init_boot`**, one of Android's own boot images.

---

## 🧰 Step 0 — Set up the tools

Download the unlock package from [Releases](https://github.com/KiMiGuel/Cubot-KingKong-ES3-Unisoc-T615-Root/releases) — `cubot_es3_unlock_package_amd64.zip` for a normal PC (x86-64), `cubot_es3_unlock_package_arm64.zip` for ARM64 machines — unzip, and build:

```
cd cubot_es3_unlock_package
make
gcc gen_spl-unlock.c -o gen_spl-unlock
gcc chsize.c -o chsize
```

Verify you built the right thing — both checks must pass:

```
./spd_dump --help          # must NOT list a "baudrate" command
grep reopen_port common.c  # must find the reconnect fix
```

### 📦 Where every file comes from

| File | Where it comes from |
|---|---|
| `spd_dump`, `gen_spl-unlock`, `chsize` | You just built them from the package source (commands above) |
| `fdl1-dl.bin`, `fdl2-dl.bin`, `fdl2-cboot.bin`, `misc-wipe.bin`, `custom_exec_no_verify_65015f08.bin` | Ship **ready-made inside the package** — nothing to build, don't rename or move them |
| `splloader.bin`, `uboot.bin` | You dump them off your own phone in Step 1 |
| `spl-unlock.bin` | `gen_spl-unlock` creates it from your dumped `splloader.bin` (Step 1) — it's your splloader with the lock check neutered |
| `u-boot-spl-16k-sign.bin`, `uboot_bak.bin` | Your renamed backups from Step 1 — used to restore the phone in Step 5 |
| `init_boot.img` | **NOT in the package** — you extract it from your own stock PAC (below) |
| `magisk_patched-*.img` | The Magisk app creates it on the phone (Step 6) |

### 📥 Getting `init_boot.img` out of the stock PAC

A `.pac` file is Unisoc's all-in-one firmware container — one big file holding every partition image. You need exactly one image out of it.

1. Download the official firmware zip for the build at the top of this guide from Cubot's own support page: [cubot.net/support](https://cubot.net/Support/id/135/cid/27.html) → KingKong ES 3 → `CUBOT_KINGKONG_ES_3_F071_V16_20260309`. Unzip it — inside is the `.pac` file (~3 GB).
2. Extract the PAC on Linux:

```
git clone https://github.com/bismoy-bot/PAC-Extractor
cd PAC-Extractor
python3 extractor.py /path/to/your/firmware.pac extracted
```

3. Inside `extracted/`, find `init_boot.img` (if it's named `init_boot_a.img`, rename it) and copy it into the package folder.

Stay inside the package folder the whole time — dumps land in the current directory.

---

## 💾 Step 1 — Back up & patch your loaders

Enter **BROM**: power off → hold **Volume Down** → plug USB (or press Power) → keep holding. If `1782:4d00` doesn't appear, retry with **Volume Up** — the working key has been seen to change with device state. Confirm:

```
lsusb | grep -i '1782:4d00'
```

Dump `splloader` + `uboot` (copy them off the phone into files in the current folder):

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r splloader r uboot e splloader e splloader_bak reset
```

What this does: opens BROM with the exploit → loads FDL1+FDL2 → reads `splloader` and `uboot` into `splloader.bin` / `uboot.bin` → erases the splloader slots (they get rewritten in Step 5) → reboots.

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

What this step does: temporarily replaces the `uboot` partition with `fdl2-cboot.bin`, a cooperative copy of FDL2 — the stock uboot would refuse to take part in what comes next. Your original uboot is safely backed up and returns in Step 5.

Re-enter BROM (Step 1), then:

```
spd_dump --wait 300 exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec w uboot fdl2-cboot.bin reset
```

Wait ~10 seconds after the reset.

---

## 🔓 Step 3 — Run the unlock

What this step does: sends the neutered `spl-unlock.bin` up through BROM and runs it. It's the splloader code **with the lock check removed** — and the mere act of it running flips the unlock flag in `miscdata`. That flag flip IS the unlock; there is no "restore" dance afterwards.

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

You don't trust what the tool printed — you read the flag itself out of `miscdata`:

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec verbose 2 read_part miscdata 8192 64 m.bin reset
```

Check `m.bin`: 64 zero bytes = still locked (repeat Step 3) · 32-byte string + two 16-byte hashes = **unlocked** ✅. A trailing timeout here is normal.

---

## 🧹 Step 5 — Restore & wipe

What this step does: puts the phone back together — writes your original `splloader` and `uboot` backups back (the neutered splloader and `fdl2-cboot` were temporary guests only), and writes `misc-wipe.bin` to force the factory reset that an unlock requires.

```
spd_dump exec_addr 0x65015f08 fdl fdl1-dl.bin 0x65000800 fdl fdl2-dl.bin 0x9efffe00 exec r boot w splloader u-boot-spl-16k-sign.bin w uboot uboot_bak.bin w misc misc-wipe.bin reset
```

Let the phone factory-reset and boot. 🟠 **Orange/unlocked warning screen = success.**

---

## 🪄 Step 6 — Root with Magisk

The bootloader is unlocked — now root is just Magisk patching `init_boot.img` (that's why you extracted it in Step 0) and flashing the patched copy. Done in **fastbootd** (userspace fastboot), not the plain bootloader.

Boot Android → Settings → About Phone → tap Build Number ×7 → Developer Options → **USB debugging**. Then:

```
adb push init_boot.img /sdcard/Download/
```

In the Magisk app: **Install → Select and Patch a File → `init_boot.img`**, pull the patched image back, and flash it — in **fastbootd**, not the plain bootloader:

```
adb pull /sdcard/Download/magisk_patched-*.img .
mv magisk_patched-*.img init_boot_a.img
adb reboot fastboot                 # goes STRAIGHT to fastbootd — skips the bootloader entirely.
                                    # On-device confirmation: screen looks like Android Recovery —
                                    # "FASTBOOTD" in yellow at the top, red lettering below.
fastboot devices                    # must show "<serial> fastbootd" — confirm before flashing
fastboot flash init_boot_a init_boot_a.img
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
- [`REPORT_INDEX.md`](REPORT_INDEX.md) — stock-vs-live partition comparison series
- [`UNLOCK_ROOT_RESEARCH.md`](UNLOCK_ROOT_RESEARCH.md) — the story behind the unlock: how root was achieved, in plain language

This repo stores **docs and hashes only** — no firmware images, PACs, partition dumps, or `.ko` files. It documents one device/build state; don't treat it as a universal rooting recipe.

🙏 Built on [TomKing062's CVE-2022-38694 toolkit](https://github.com/TomKing062/CVE-2022-38694_unlock_bootloader) (`spd_dump`/`spreadtrum_flash`, `gen_spl-unlock`, `chsize`), [PAC-Extractor](https://github.com/bismoy-bot/PAC-Extractor) by Bismoy Ghosh, and [Magisk](https://github.com/topjohnwu/Magisk) by topjohnwu.
