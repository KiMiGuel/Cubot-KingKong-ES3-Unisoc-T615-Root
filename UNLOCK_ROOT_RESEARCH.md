# Cubot KingKong ES 3 — Unlock & Root Research

**How a phone with no fastboot unlock, no recovery partition, and zero community support ended up fully rooted — on its completely stock kernel.**

**Device:** Cubot KingKong ES 3 · Unisoc T615 (UMS9230_6h10) · UFS · A/B slots
**Verified build:** `CUBOT_KINGKONG_ES_3_F071_V16_20260309`

This is the readable version of the project record. Every claim below is backed by captured evidence in the companion documents (`BOOT_CHAIN_FINDINGS.md`, `ROOTING_GUIDE.md`, `REPORT_INDEX.md`) — this file tells the story; those files prove it.

---

## The phone

The KingKong ES 3 is a budget rugged phone: 6 GB RAM, 256 GB of UFS storage, a 90 Hz screen, and a Unisoc T615 chip running an Android 15 userspace on top of an Android-13 GKI kernel branch. Nobody buys this phone to mod it, and it shows — when this project started, there was no unlock method, no root guide, no community thread, nothing. Everything that follows had to be established from scratch, on the device itself.

## Three dead ends and one door

Two facts kill entire categories of advice you'll find elsewhere:

- **There is no fastboot unlock.** `fastboot flashing unlock` returns `Unrecognized command`. The OEM simply never wired it up.
- **There is no recovery partition.** The GPT has 74 entries and not one of them is named `recovery`. Any guide that starts with "boot into recovery" is dead on arrival here.

The door is **BROM** — the BootROM, a tiny loader burned into the chip itself. Power off, hold Volume Down, plug in USB, and the phone enumerates as `1782:4d00` before a single byte of Android or even the bootloader has run. BROM can't be bricked, corrupted, or locked out, because it isn't stored anywhere writable. Everything in this project goes through that door.

The lock itself was defeated with **CVE-2022-38694** — a published signature-check bypass in Unisoc's FDL1 download stage, with a public toolkit by TomKing062. No new vulnerability was needed; the work was making the existing one function on this specific device, from Linux, reliably.

## The unlock, in plain terms

The chain, end to end:

1. Through BROM, load Unisoc's own download agents (FDL1/FDL2) and **dump the phone's actual bootloaders** (`splloader`, `uboot`) — your backups, straight from your device.
2. Generate the unlock payload: the phone's *own* `splloader` with the lock-state check patched out — 37 bytes changed across a handful of instruction clusters. Not shellcode; the real loader, minus the part that refuses.
3. Stage a working FDL2 in place of `uboot`, then **run the patched loader once**. The lock check never executes, and the bootloader's unlock flag flips.
4. **Verify by reading, not by trusting**: read back the `miscdata` partition. Sixty-four zero bytes means still locked; a 32-byte string plus two 16-byte hashes means unlocked.
5. **Restore everything**: original loaders back in place, a wipe flag written, phone factory-resets itself and boots — with the orange "unlocked" warning screen that is, in this context, a certificate of achievement.

Magisk root then lands the standard modern way: patch the stock `init_boot.img` in the Magisk app, flash it to `init_boot_a` from fastbootd. `boot_a` is never touched. The running kernel remains 100% stock.

## The tool that had to be rebuilt first

The upstream `spd_dump` is a Windows-first codebase, and its Linux path did not work reliably against this device. So the project rebuilt it: every `_WIN32`/WinUSB code path stripped out, leaving a clean Linux/libusb-only tool, verified with a zero-warnings build.

That exposed the real bug. Mid-handshake — every time — the phone would **vanish from the USB bus**. Live `lsusb` polling showed what was actually happening: the device disconnects, briefly re-enumerates as "Gadget Serial," drops again, and comes back as the expected Unisoc diag interface four to five seconds later. The re-enumeration is triggered by the unlock payload itself. It's normal. It's supposed to happen.

The code disagreed. Two independent "device gone" handlers existed: the CHECK_BAUD loop's own retry logic (correct, patient, designed for exactly this) and a separate unconditional kill-switch in the USB I/O layer that fired first, every time, on the very next call after the disconnect. The patient logic never got a turn.

The fix was a shared `reopen_port()` helper — up to 30 reconnect attempts, one second apart — wired into both guards. On real hardware, the very next run caught the disconnect twice in one session and sailed past the point where every previous attempt had died. The device was never the problem; the tool was.

## The success message that looks like a crash

Here's the part that confuses everyone, including this project at first: after the unlock payload runs, the tool prints a successful `CMD_CONNECT FDL1` handshake — and then throws a **timeout**. It looks like failure. It is, in fact, exactly what success looks like.

The reason is architectural. The patched lock check lives in the loader's early init code, and the BSL handshake only completes *after* that code has run — so by the time `CMD_CONNECT FDL1` prints, the unlock has already been applied. The one-line invocation has also fully consumed its command list by then; the tool drops into an interactive prompt with nothing left to say, and the next read simply times out. Nothing is broken. Nothing is listening. Re-running because "it errored" is how people talk themselves into thinking the unlock failed when it had already worked.

The project rule that came out of this: **state is proven by reading, never by trusting tool output.** The `miscdata` readback in Step 4 of the guide exists precisely because the tool's own output can't be taken at face value.

## The worst day — and why it should encourage you

One day before the final unlock, the device went through its worst event of the project. An interrupted restore session left the bootloader relocked and the OS erased. The phone booted to a stark "warning: no OS" screen and then fell back into BROM on every single boot — because `uboot` was still holding the staged FDL2 from the unlock work, so there was no normal bootloader to hand off to.

"Warning: no OS" turned out not to be one bug. Four suspects were eliminated in order, each with evidence: vbmeta verification state — ruled out; a stale recovery boot-command in `misc` — ruled out; GPT corruption — ruled out, all eight saved partition maps from six weeks of work were byte-identical; a userdata wipe — changed nothing. What cleared it was unglamorous: stock `uboot` back in place, the same unlock chain re-run, both slots reflashed with hash-verified images. The phone came back, and was rooted the next day.

This section exists for one reason: **at no point in the brick was BROM unreachable.** Volume Down + USB answered every time. If you soft-brick this phone mid-process, you are in exactly the position this device was in — and this device came back.

## What "rooted" actually means here

The live boot chain, audited partition by partition against the stock firmware:

| Component | State |
|---|---|
| `boot_a` | Stock, byte-for-byte |
| `vendor_boot_a` | Stock, byte-for-byte |
| `dtbo_a` | Stock, byte-for-byte |
| `init_boot_a` | Magisk-patched — the root carrier |
| `vbmeta_a` | One flags byte changed; all descriptors stock |
| Bootloader | Unlocked (orange state) |

Root comes entirely from the patched `init_boot_a`. And to be precise about claims: a rooted phone is proof that root works — nothing more. This project makes no claim beyond what is listed in this table.

## If you attempt this: the rules this project runs on

- **Hash every image before and after use.** Never identify an image by its filename.
- **In `spd_dump`, never run** `repartition`, `erase_all`, or `ufs repart`/`rawerase`/`setattr` — they can truncate the GPT and turn a soft brick into a real problem.
- **`prodnv` / `nvitem` / `l_fixnv` hold factory calibration data.** Hands off — permanently.
- USB 2.0 port and cable only; USB 3.0 is unstable with this device in fastbootd and flaky in FDL sessions.
- Use Linux. Stop ModemManager before you start (`LIBUSB_ERROR_BUSY` is ModemManager grabbing the diag port).
- Start `spd_dump` first, *then* plug the phone — not the other way around.

## Credits

- **TomKing062** — author of `spd_dump`/`spreadtrum_flash`, `gen_spl-unlock`, `chsize`, and the CVE-2022-38694 unlock toolkit this chain is built on. The Linux/libusb-only rewrite and the `reopen_port()` reconnect fix were developed independently for this device.
- **topjohnwu** — Magisk, used for the `init_boot` root patch.
- **libusb** — the USB transport the rewritten `spd_dump` runs on.
- **KiMiGuel (INDEPENTEST)** — device adaptation, tool rewrite and debug, brick recovery, boot-chain analysis, and the verification methodology described above.
