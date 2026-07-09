# Controlled Boot Image Comparison

Scope: boot image only. No original images were modified. No flashing, patching, repacking, resigning, or new boot image generation was performed. Extracted artifacts were written under `./comparison/boot_3way/`.

## A. Executive Summary

The live active slot A boot image is an exact byte-for-byte match for the stock PAC `boot-gki.img`.

The live boot image does not match the custom candidate and does not match the custom Magisk candidate. Normalized unpacked payloads show the same result:

- Stock PAC and live active slot A have the same extracted kernel hash and empty boot ramdisk.
- Custom candidate and custom+Magisk candidate share a different custom kernel hash.
- Custom+Magisk candidate additionally contains a non-empty Magisk-style boot ramdisk.

Conclusion: active slot A boot currently looks stock for `boot_a`. There is no evidence in the active live `boot_a` image that the custom kernel or custom Magisk boot image was placed into the active boot path at the time this dump was taken.

## B. Side-by-Side View

| Field | Stock PAC | Live Active Slot A | Custom Candidate | Custom+Magisk Candidate |
|---|---:|---:|---:|---:|
| Source path | `/home/kali/pacextractor/pac_out/boot-gki.img` | `../slot_a_ordered/02_boot_a_live.img` | `/home/kali/Kernels/CANDIDATE/boot_b.img` | `/home/kali/Kernels/CANDIDATE/boot_b_magisk.img` |
| Raw size | 67,108,864 bytes | 67,108,864 bytes | 34,181,120 bytes | 34,545,664 bytes |
| Raw SHA256 | `3113ec9523f41a8a667c594c8590731876233b82d80c9c13a551f697f240f1da` | `3113ec9523f41a8a667c594c8590731876233b82d80c9c13a551f697f240f1da` | `de91093263b7c229c8efea1cb4fa7d1dc1ed5bdfd3238f05aeb8fd944bfd91af` | `7670cfb93a8b53e78aa2af4e99994500312bc57b28ddf69f6f2b2f0e4424e8a2` |
| `file` output | Android bootimg, kernel | Android bootimg, kernel | Android bootimg, kernel | Android bootimg, kernel `(0x58188)` |
| Header version | 4 | 4 | 4 | 4 |
| Kernel size | 47,442,432 bytes | 47,442,432 bytes | 34,175,488 bytes | 34,175,488 bytes |
| Ramdisk size | 0 bytes | 0 bytes | 0 bytes | 360,840 bytes |
| Cmdline | empty | empty | empty | empty |
| OS version | None in boot header; AVB prop says `13` | None in boot header; AVB prop says `13` | None | None |
| OS patch level | None in boot header; AVB prop says `2025-04-05` | None in boot header; AVB prop says `2025-04-05` | None | None |
| Page size | Not present in v4 boot header output | Not present in v4 boot header output | Not present in v4 boot header output | Not present in v4 boot header output |
| DTB extracted from boot | absent | absent | absent | absent |
| Bootconfig extracted from boot | absent | absent | absent | absent |
| AVB footer | present | present | not detected | not detected |

## C. Raw Image Comparison

| Pair | Exact raw match |
|---|---:|
| Stock PAC vs Live Active Slot A | yes |
| Live Active Slot A vs Custom Candidate | no |
| Live Active Slot A vs Custom+Magisk Candidate | no |
| Stock PAC vs Custom Candidate | no |
| Custom Candidate vs Custom+Magisk Candidate | no |

Raw SHA256 values are also saved in:

- `STOCK_LIVE_CUSTOM_BOOT_SHA256SUMS.txt`
- `comparison/boot_3way/RAW_IMAGE_SHA256SUMS.txt`

Stock and live are both 64 MiB partition-sized images and have identical raw hashes. The custom images are smaller boot-image payloads and are not 64 MiB partition-sized images.

## D. Normalized Payload Comparison

| Field | Stock PAC | Live Active Slot A | Custom Candidate | Custom+Magisk Candidate |
|---|---:|---:|---:|---:|
| Kernel SHA256 | `8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532` | `8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532` | `773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a` | `773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a` |
| Ramdisk status | empty | empty | empty | present |
| Ramdisk SHA256 | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` | `e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855` | `d090b82429d1a191befd5ba87a10ba06419f4fdbbf66b9f3e5592dee6f2df2c1` |
| Header metadata relationship | same as live | same as stock | same header version, different kernel size | same custom kernel plus ramdisk |
| Likely relationship to live boot | exact source match | active image | not active live boot | not active live boot |

The custom+Magisk ramdisk stream lists Magisk-related entries:

```text
.backup/.magisk
overlay.d/sbin/magisk.xz
overlay.d/sbin/stub.xz
```

Live active slot A has an empty boot ramdisk, matching stock PAC and not matching the Magisk candidate.

## E. Interpretation

Does live boot match stock exactly?

Yes. The stock PAC boot image and live active slot A boot image have the same raw SHA256 and `cmp` reports an exact match.

Does live boot appear Magisk-patched?

No. The live boot image has an empty boot ramdisk and exactly matches the stock PAC image. The custom Magisk image has a non-empty LZ4 ramdisk with Magisk/overlay entries.

Does live boot contain the custom candidate kernel?

No. The live extracted kernel hash is `8b9584d8518c79ab80c114143e1190b7e8e9058964adc8cffde98c53f083a532`. The custom candidate kernel hash is `773d9adcb8c20f954a584fcc84b147452028074c148b82fb4983bf2b3dee132a`.

Is the custom size mismatch explained by lack of 64 MiB partition padding?

Partly, yes. The custom candidates are compact Android boot images, not 64 MiB partition-sized images with stock-style padding and AVB footer. However, the normalized comparison also shows a real payload difference: the custom kernel is different and smaller than the stock/live kernel.

Is there evidence the custom boot was never placed into the active boot path?

For the dumped active slot A boot path, yes. The live `boot_a` dump is byte-for-byte identical to stock PAC and has no custom kernel or Magisk ramdisk. This does not prove what may have happened in another slot or at another time, but it does show the active `boot_a` image analyzed here is stock.

## F. Clear Conclusion

| Comparison | Result |
|---|---|
| Stock vs Live | exact raw match and exact normalized payload match |
| Live vs Custom | different raw image, different kernel payload |
| Live vs Custom Magisk | different raw image, different kernel payload, live lacks Magisk ramdisk |
| Stock vs Custom | different raw image, different kernel payload |

Overall: live active slot A `boot_a` is stock PAC `boot-gki.img`, not the custom candidate and not the custom Magisk candidate.

## G. Next Recommended Step

Stop after this boot-only comparison.

The next analysis step should be to expand the same controlled raw and normalized comparison method to the rest of the active boot chain:

- `vendor_boot`
- `init_boot`
- `dtbo`
- `vbmeta`

Do not treat this report as a flashing recommendation.

## Artifacts

Unpacked comparison artifacts:

```text
comparison/boot_3way/stock_boot_gki/
comparison/boot_3way/live_boot_a/
comparison/boot_3way/custom_boot_b/
comparison/boot_3way/custom_boot_b_magisk/
```

Supporting command-output files:

```text
comparison/boot_3way/RAW_IMAGE_FILE_OUTPUT.txt
comparison/boot_3way/RAW_IMAGE_SHA256SUMS.txt
comparison/boot_3way/UNPACK_BOOTIMG_METADATA.txt
comparison/boot_3way/NORMALIZED_PAYLOAD_SHA256SUMS.txt
comparison/boot_3way/EXACT_MATCH_MATRIX.txt
comparison/boot_3way/AVBTOOL_BOOT_INFO.txt
```

Tool note:

- `magiskboot` was missing from `PATH`; it was not required for this comparison because `unpack_bootimg`, `file`, `sha256sum`, `cmp`, `lz4cat`, and `cpio` were sufficient.
