# Init Boot Stock vs Live Comparison

Scope: `init_boot` only. No original images were modified. No flashing, patching, repacking, resigning, or new boot image generation was performed. Extracted artifacts were written under `./comparison/init_boot_stock_vs_live/`.

## A. Executive Summary

Live active slot A `init_boot_a` does not match stock PAC `init_boot.img`.

The difference is not just padding or partition sizing. Both images are 8 MiB partition-sized images, but their raw hashes differ, their extracted ramdisk hashes differ, and the live ramdisk contains Magisk patch artifacts.

Key findings:

- Stock and live `init_boot` both use Android boot image header version 4.
- Stock and live both have an empty kernel field.
- Stock ramdisk size: 2,606,941 bytes.
- Live ramdisk size: 2,316,314 bytes.
- Live ramdisk contains `.backup/.magisk`, `.backup/init.xz`, `.backup/.rmlist`, and `overlay.d/sbin/magisk.xz`.
- Live ramdisk `init` differs from stock `init`.
- `system/etc/init/snapuserd.rc` and `system/etc/ramdisk/build.prop` match between stock and live.

Conclusion: live active slot A `init_boot_a` appears Magisk-patched. Since the previous boot-only comparison showed `boot_a` is stock, root modifications are likely being carried by `init_boot_a`.

## B. Side-by-Side Table

| Field | Stock PAC init_boot | Live Active Slot A init_boot |
|---|---:|---:|
| Source path | `/home/kali/pacextractor/pac_out/init_boot.img` | `../slot_a_ordered/04_init_boot_a_live.img` |
| Raw size | 8,388,608 bytes | 8,388,608 bytes |
| Raw SHA256 | `7d57db8a3eb7e7544bdb18a313eec6dbe03d09a2e19dd962cc56f620904e5a92` | `4a6563154596ab31558ffbda94523c251ab2292567c1384fcffd74f9c02362b6` |
| `file` output | Android bootimg | Android bootimg |
| Header version | 4 | 4 |
| Kernel size | 0 bytes | 0 bytes |
| Ramdisk size | 2,606,941 bytes | 2,316,314 bytes |
| Ramdisk SHA256 | `793a92bb18dd886bef7d87812ccbb84234a8eaeaea2443ca28621b4f685d9184` | `098834ff153c5ddb53a996e9889b6f92dc19c83dbb0c28355fef8194ae47f878` |
| Cmdline | empty | empty |
| Boot header OS version | None | None |
| Boot header OS patch level | None | None |
| AVB init_boot OS version prop | `15` | `15` |
| Page size | Not present in v4 boot header output | Not present in v4 boot header output |
| AVB footer | present | present |
| Ramdisk cpio entries | 21 | 29 |

## C. Raw Image Comparison

| Pair | Exact raw match |
|---|---:|
| Stock PAC init_boot vs Live active slot A init_boot | no |

Raw hashes are saved in:

- `STOCK_LIVE_INIT_BOOT_SHA256SUMS.txt`
- `comparison/init_boot_stock_vs_live/RAW_IMAGE_METADATA.txt`

Both raw images are 8 MiB, so the mismatch is not explained by a 64 MiB-vs-compact image size difference like the earlier custom boot comparison. These are same-sized partition images with different content.

## D. Normalized Ramdisk Comparison

| Field | Stock PAC init_boot | Live active slot A init_boot |
|---|---:|---:|
| Kernel payload | empty | empty |
| Kernel hash | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` |
| Ramdisk payload | LZ4 cpio | LZ4 cpio |
| Ramdisk hash | `793a92bb18dd886bef7d87812ccbb84234a8eaeaea2443ca28621b4f685d9184` | `098834ff153c5ddb53a996e9889b6f92dc19c83dbb0c28355fef8194ae47f878` |
| `init` member hash | `21ebf60e3c113a84d78d9c80be7312f3d36604c77de1da8c0c66044259b1f693` | `383670a7ba3a6a4b79e5f3467e1da4b66a5df66a9b356ab9f70916854dd6b468` |
| `snapuserd.rc` hash | `3a6966f87823b2726a231e7244adf78ad91d581bf4e8b1580630ebb82cc6c498` | `3a6966f87823b2726a231e7244adf78ad91d581bf4e8b1580630ebb82cc6c498` |
| `build.prop` hash | `c917c9b7ab0308daf92108e2189b3a73d7d9aa691c0b75bbf8dfab08e9525831` | `c917c9b7ab0308daf92108e2189b3a73d7d9aa691c0b75bbf8dfab08e9525831` |

Unique live-only ramdisk entries:

```text
.backup
.backup/.magisk
.backup/.rmlist
.backup/init.xz
overlay.d
overlay.d/sbin
overlay.d/sbin/init-ld.xz
overlay.d/sbin/magisk.xz
overlay.d/sbin/stub.xz
```

There were no unique stock-only files after duplicate path normalization.

## E. Magisk / Root Evidence

Live `init_boot_a` contains direct Magisk evidence:

```text
.backup/.magisk
.backup/init.xz
.backup/.rmlist
overlay.d/sbin/init-ld.xz
overlay.d/sbin/magisk.xz
overlay.d/sbin/stub.xz
```

Live ramdisk string search also found:

```text
/.magisk
overlay.d
/init-ld.xz
/data/magisk
${MAGISKTMP}
overlay.d/sbin/stub.xz
```

Stock ramdisk did not produce Magisk-specific path/string hits in the saved search artifact. The live `init` member hash differs from stock, and live includes `.backup/init.xz`, consistent with a Magisk init replacement/backup pattern.

AVB note: both images contain an AVB footer using public key SHA1 `575d2803b0e6cfb05ff6175a97ccbe688c7260ff`. The live image's `avbtool info_image` output still shows the same init_boot fingerprint and hash-descriptor values as stock, while the raw and ramdisk payloads differ. This is additional evidence that the live image has been modified from the stock payload.

## F. Clear Conclusion

Does live init_boot match stock exactly?

No. The raw image hashes differ, `cmp` reports no raw exact match, and the normalized ramdisk hashes differ.

Does live init_boot appear Magisk-patched?

Yes. The live ramdisk contains Magisk backup and overlay files, Magisk strings, and a modified `init` member compared with stock.

Is root likely coming from init_boot?

Yes. Given that the previous boot-only comparison showed live `boot_a` is byte-for-byte stock, the live root/Magisk evidence found here points to `init_boot_a` as the likely carrier for root modifications.

## G. Next Recommended Step

Stop after this `init_boot` comparison.

Because live `init_boot_a` differs from stock, the next analysis step should be a controlled stock-vs-live comparison of:

- `vendor_boot`
- `dtbo`

Do not treat this report as a flashing recommendation.

## Artifacts

Unpacked comparison artifacts:

```text
comparison/init_boot_stock_vs_live/stock_init_boot/
comparison/init_boot_stock_vs_live/live_init_boot_a/
```

Supporting command-output files:

```text
comparison/init_boot_stock_vs_live/RAW_IMAGE_METADATA.txt
comparison/init_boot_stock_vs_live/EXACT_MATCH_MATRIX.txt
comparison/init_boot_stock_vs_live/UNPACK_BOOTIMG_METADATA.txt
comparison/init_boot_stock_vs_live/NORMALIZED_PAYLOAD_SHA256SUMS.txt
comparison/init_boot_stock_vs_live/AVBTOOL_INIT_BOOT_INFO.txt
comparison/init_boot_stock_vs_live/MKBOOTIMG_ARGS.txt
comparison/init_boot_stock_vs_live/MAGISK_ROOT_STRING_HITS.txt
comparison/init_boot_stock_vs_live/RAMDISK_SELECTED_MEMBER_HASHES.txt
comparison/init_boot_stock_vs_live/RAMDISK_STOCK_ONLY_FILES.txt
comparison/init_boot_stock_vs_live/RAMDISK_LIVE_ONLY_FILES.txt
comparison/init_boot_stock_vs_live/RAMDISK_BUILD_PROP.txt
```
