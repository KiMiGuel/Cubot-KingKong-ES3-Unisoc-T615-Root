# DTBO Stock vs Live Comparison

Scope: `dtbo` only. No original images were modified. No flashing, patching, repacking, resigning, or new image generation was performed. Extracted artifacts were written under `./comparison/dtbo_stock_vs_live/`.

## A. Executive Summary

Live active slot A `dtbo_a` is an exact byte-for-byte match for stock PAC `dtbo.img`.

The normalized DTBO comparison also matches:

- DTBO table metadata matches after excluding only the source image path line.
- Both images contain 13 DTBO entries.
- All 13 extracted DTBO entry hashes match.
- All 13 decompiled DTS hashes match.
- All requested UFS/regulator search counts match between stock and live.

Conclusion: there is no evidence that live `dtbo_a` was modified during rooting. DTBO content confirms UFS-related overlay paths and regulator-related overlay bindings, but the live copy is stock.

## B. Side-by-Side Table

| Field | Stock dtbo | Live dtbo_a |
|---|---:|---:|
| Source path | `/home/kali/pacextractor/pac_out/dtbo.img` | `../slot_a_ordered/01_dtbo_a_live.img` |
| Raw size | 8,388,608 bytes | 8,388,608 bytes |
| Raw SHA256 | `f16238c3ddc31dc5015d42523d29e5fa74ca2b3b43df6fbaed9aad11c18f5672` | `f16238c3ddc31dc5015d42523d29e5fa74ca2b3b43df6fbaed9aad11c18f5672` |
| `file` output | data | data |
| DTBO magic | `0xd7b7ab1e` | `0xd7b7ab1e` |
| DTBO total size | 1,462,504 bytes | 1,462,504 bytes |
| Header size | 32 bytes | 32 bytes |
| Entry size | 32 bytes | 32 bytes |
| Entry count | 13 | 13 |
| Entries offset | 32 | 32 |
| Page size | 2,048 | 2,048 |
| Version | 0 | 0 |
| AVB footer | present | present |

## C. Raw Image Comparison

| Pair | Exact raw match |
|---|---:|
| Stock PAC dtbo vs Live active slot A dtbo | yes |

Raw hashes are saved in:

- `STOCK_LIVE_DTBO_SHA256SUMS.txt`
- `comparison/dtbo_stock_vs_live/RAW_IMAGE_METADATA.txt`

`avbtool info_image` also reports matching DTBO AVB metadata:

- Original image size: 1,462,504 bytes
- VBMeta offset: 1,466,368
- VBMeta size: 2,112 bytes
- Public key SHA1: `98e711e004cbcad564b5692c88044726e92d631f`
- Hash descriptor digest: `f99cb033de3af0669d82156176bfd36bff4eefef2ba858236fbd4fa7e8e5eac9`

## D. Normalized DTBO Comparison

| Field | Result |
|---|---|
| Raw image exact match | yes |
| Normalized DTBO table metadata match | yes |
| DTBO entry count | 13 stock / 13 live |
| Extracted entry hash match | yes, all 13 |
| DTS decompile status | OK for all 13 stock and all 13 live |
| Decompiled DTS hash match | yes, all 13 |
| Exact normalized match | yes |

DTBO entry hash table:

| Entry | Stock DTBO SHA256 | Live DTBO SHA256 | Match | Size |
|---|---|---|---:|---:|
| 00 | `8ca49b00e5859a658fbf59e9a1f867fbcbee98ea15cbc33048f0e4fe9136792b` | `8ca49b00e5859a658fbf59e9a1f867fbcbee98ea15cbc33048f0e4fe9136792b` | yes | 124,916 |
| 01 | `79882063aff448d84fdc705007eab724a4f0273bd25224ef3d42c291dbf99b96` | `79882063aff448d84fdc705007eab724a4f0273bd25224ef3d42c291dbf99b96` | yes | 125,441 |
| 02 | `8c910aca45453e42ceaab78037c4c41b198261b6e038beb5a9a152d95f792fd4` | `8c910aca45453e42ceaab78037c4c41b198261b6e038beb5a9a152d95f792fd4` | yes | 124,916 |
| 03 | `961717a8ee6492aa97c98436ba0a603da6892170613f2e6ae04cef8f8e298070` | `961717a8ee6492aa97c98436ba0a603da6892170613f2e6ae04cef8f8e298070` | yes | 125,441 |
| 04 | `0214a031ca5a575d515262df7ca2b57b1c90fb65b379992bbc2375159bf80705` | `0214a031ca5a575d515262df7ca2b57b1c90fb65b379992bbc2375159bf80705` | yes | 39,591 |
| 05 | `9206312336cc8df5c352332f7c2aaec8f45f4bb1800ad2ed7d4eaf0fb0ef5710` | `9206312336cc8df5c352332f7c2aaec8f45f4bb1800ad2ed7d4eaf0fb0ef5710` | yes | 126,202 |
| 06 | `a90ae5969cafcff0220f3ae5aa052b14c7ac1017b26e2c4fd9d0c2280fdf25ff` | `a90ae5969cafcff0220f3ae5aa052b14c7ac1017b26e2c4fd9d0c2280fdf25ff` | yes | 125,665 |
| 07 | `f7b14f2b3d178c77c407e145632b5a810a02bc1307627803ee45a41c6ded876c` | `f7b14f2b3d178c77c407e145632b5a810a02bc1307627803ee45a41c6ded876c` | yes | 119,803 |
| 08 | `b5a1906507d8fcc0fdeac5aba32012076d7c3a76f3428c2485f892df0b0e39e1` | `b5a1906507d8fcc0fdeac5aba32012076d7c3a76f3428c2485f892df0b0e39e1` | yes | 121,714 |
| 09 | `27509d9e7fa37e38fc2728c94c01f49b3d915d8997f5e9abff9d2d071db74a1a` | `27509d9e7fa37e38fc2728c94c01f49b3d915d8997f5e9abff9d2d071db74a1a` | yes | 122,266 |
| 10 | `5988498572ee941054d5d782e60748e79e7c51ce7a5b1d23d5a15267f10dc42a` | `5988498572ee941054d5d782e60748e79e7c51ce7a5b1d23d5a15267f10dc42a` | yes | 124,916 |
| 11 | `7b2a21d5e7452c61e1c3a5648d8122e98ba85e0fb3234791b1ddfc1433551c58` | `7b2a21d5e7452c61e1c3a5648d8122e98ba85e0fb3234791b1ddfc1433551c58` | yes | 54,022 |
| 12 | `bb26c816f364ff0627273796c6a880bf2e3f04b73a031f44c41cbddc2860e454` | `bb26c816f364ff0627273796c6a880bf2e3f04b73a031f44c41cbddc2860e454` | yes | 127,163 |

The full DTS hash table is saved in:

```text
comparison/dtbo_stock_vs_live/DTS_HASH_TABLE.md
```

All decompiled DTS hashes match stock.

## E. UFS / Regulator Evidence

Requested-pattern search counts:

| Pattern | Stock DTS hits | Live DTS hits | Match |
|---|---:|---:|---:|
| `20200000.ufs` | 33 | 33 | yes |
| `ufs` | 33 | 33 | yes |
| `sc2730` | 0 | 0 | yes |
| `regulator` | 28 | 28 | yes |
| `vcc` | 0 | 0 | yes |
| `vddgen0` | 0 | 0 | yes |
| `avdd12` | 8 | 8 | yes |
| `avdd18` | 0 | 0 | yes |
| `vddcore` | 0 | 0 | yes |
| `vddmodem` | 0 | 0 | yes |
| `sprd` | 3,151 | 3,151 | yes |
| `ums9230` | 32 | 32 | yes |
| `sda` | 0 | 0 | yes |
| `first_stage` | 0 | 0 | yes |
| `bootargs` | 13 | 13 | yes |
| `androidboot` | 0 | 0 | yes |

UFS references include repeated modem path overlays:

```text
sprd,file-name-ufs = "/dev/block/platform/soc/soc:ap-apb/20200000.ufs/by-name/wcnmodem";
sprd,file-name-ufs = "/dev/block/platform/soc/soc:ap-apb/20200000.ufs/by-name/gnssmodem";
sprd,file-name-ext-ufs = "/dev/block/platform/soc/soc:ap-apb/20200000.ufs/by-name/gnssmodem";
```

Regulator-related references include:

```text
regulator-name = "vddvbus";
regulator-always-on;
avdd12-supply = <0xffffffff>;
avdd12-parent-bound-chip = <0x1a4c 0x01 0x00 0x01>;
avdd12-bound-wbreq = <0x1ac8 0x1000 0x00 0x1000>;
vddwcn = "/fragment/__overlay__/sprd-marlin3:avdd12-supply:0";
pmic_regulator = "/fragment@2:target:0";
```

Direct `sc2730`, `vddgen0`, `avdd18`, `vddcore`, and `vddmodem` strings were not found in the decompiled DTBO overlays. Those strings were present in the stock/live vendor_boot base DTB report, while DTBO contributes overlay-specific UFS paths and AVDD12/WCN/regulator bindings.

Stock and live evidence matches exactly for all requested DTBO search terms.

## F. Clear Conclusion

Does live dtbo_a match stock exactly?

Yes. The stock PAC `dtbo.img` and live active slot A `dtbo_a` have identical raw SHA256 values and `cmp` reports an exact raw match.

If not exact, what differs?

Nothing was found to differ. Raw image, AVB footer metadata, normalized DTBO table metadata, all 13 entry hashes, all 13 decompiled DTS hashes, and all requested search-count evidence match.

Is there any evidence dtbo_a was modified during rooting?

No. The live `dtbo_a` is byte-for-byte stock. The rooting evidence remains isolated to `init_boot_a` from the previous comparison.

Does DTBO support the theory that custom kernel failure involved UFS/regulator/device-tree expectations?

Yes, as a compatibility-risk signal. DTBO confirms overlay references to UFS-backed modem partitions under `20200000.ufs` and regulator-related overlay bindings such as `vddvbus`, `avdd12-supply`, `vddwcn`, and `pmic_regulator`. Combined with the stock vendor_boot module evidence, this supports the theory that a custom kernel must match the stock UFS/regulator/device-tree expectations and module ABI. This report does not prove a runtime failure cause, but it keeps UFS/regulator/device-tree mismatch as a plausible custom-kernel failure mechanism.

## G. Next Recommended Step

Stop after this `dtbo` comparison.

The next analysis step should be a controlled stock-vs-live `vbmeta` comparison.

Do not treat this report as a flashing recommendation.

## Artifacts

Unpacked/decompiled comparison artifacts:

```text
comparison/dtbo_stock_vs_live/stock_dtbo/
comparison/dtbo_stock_vs_live/live_dtbo_a/
```

Supporting command-output files:

```text
comparison/dtbo_stock_vs_live/RAW_IMAGE_METADATA.txt
comparison/dtbo_stock_vs_live/EXACT_MATCH_MATRIX.txt
comparison/dtbo_stock_vs_live/DTBO_TABLE_METADATA.txt
comparison/dtbo_stock_vs_live/DTBO_ENTRY_HASH_TABLE.md
comparison/dtbo_stock_vs_live/DTS_HASH_TABLE.md
comparison/dtbo_stock_vs_live/DTC_DECOMPILE_STATUS_TABLE.md
comparison/dtbo_stock_vs_live/DTBO_SEARCH_COUNT_TABLE.md
comparison/dtbo_stock_vs_live/AVBTOOL_DTBO_INFO.txt
comparison/dtbo_stock_vs_live/stock_dtbo/DTBO_UFS_REGULATOR_TARGETED_HITS.txt
comparison/dtbo_stock_vs_live/live_dtbo_a/DTBO_UFS_REGULATOR_TARGETED_HITS.txt
```

Tool note:

- `mkdtimg` and `dump_dtbo` were missing from `PATH`; direct DTBO table parsing was used, and `dtc` successfully decompiled all 13 entries.
