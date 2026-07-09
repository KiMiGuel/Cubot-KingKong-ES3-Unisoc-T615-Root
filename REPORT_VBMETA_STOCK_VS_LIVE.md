# Controlled Stock vs Live vbmeta Comparison

## A. Executive Summary

This comparison checks only top-level `vbmeta`.

Result: live `vbmeta_a` does not match stock byte-for-byte. Both images are the same size and contain the same AVB descriptors, keys, rollback index, algorithm, release string, and build properties, but the live image has a one-byte change in the signed AVB header flags field.

The stock PAC image reports `Flags: 0`. The live active slot A image reports `Flags: 33554432`. `cmp` shows exactly one differing byte at byte 121, corresponding to the AVB header flags field. `avbtool verify_image` can verify the stock vbmeta struct signature, while live `vbmeta_a` fails signature verification.

Interpretation: live `vbmeta_a` is modified, but the modification does not change chain descriptors or partition identity metadata. The current root chain is best described as stock boot/vendor_boot/dtbo, patched `init_boot`, modified-but-descriptor-identical `vbmeta_a`, and unlocked/orange LK behavior that skips verification.

## B. Side-by-Side Table

| Field | Stock `vbmeta-sign.img` | Live `vbmeta_a` |
|---|---|---|
| Path | `/home/kali/pacextractor/pac_out/vbmeta-sign.img` | `../slot_a_ordered/00_vbmeta_a_live.img` |
| Size | 1,048,576 bytes | 1,048,576 bytes |
| SHA-256 | `a30d4d8bc891320613f973153bf226fb1e580d9943fa1bc65025282584fbb6b5` | `6935f498ac76f8f175e64f3d51fa6db8aa398cdf2ac1d5b0a190ad2dbca74715` |
| `file` output | `data` | `data` |
| Minimum libavb version | 1.0 | 1.0 |
| Header block | 256 bytes | 256 bytes |
| Authentication block | 576 bytes | 576 bytes |
| Auxiliary block | 17,472 bytes | 17,472 bytes |
| Algorithm | `SHA256_RSA4096` | `SHA256_RSA4096` |
| Public key SHA-1 | `7b595403a7796dc3c96d57d79583a21721893d20` | `7b595403a7796dc3c96d57d79583a21721893d20` |
| Rollback index | 0 | 0 |
| Rollback index location | 0 | 0 |
| Flags | `0` | `33554432` |
| Release string | `avbtool 1.3.0` | `avbtool 1.3.0` |
| Public key metadata | none, size 0 | none, size 0 |
| AVB descriptor set | Matches live except top-level flags | Matches stock except top-level flags |
| Signature check | vbmeta struct signature verified | signature check failed |

Artifacts:
- `comparison/vbmeta_stock_vs_live/RAW_IMAGE_METADATA.txt`
- `comparison/vbmeta_stock_vs_live/STOCK_AVBTOOL_INFO.txt`
- `comparison/vbmeta_stock_vs_live/LIVE_AVBTOOL_INFO.txt`
- `comparison/vbmeta_stock_vs_live/AVBTOOL_INFO_DIFF.txt`
- `comparison/vbmeta_stock_vs_live/AVBTOOL_VERIFY_IMAGE.txt`
- `comparison/vbmeta_stock_vs_live/DESCRIPTOR_PARTITION_COVERAGE.md`

## C. Raw Image Comparison

| Check | Result |
|---|---|
| Size match | Yes |
| SHA-256 match | No |
| Exact raw byte match | No |
| Number of differing bytes | 1 |
| First and only difference | byte 121: stock `0`, live `2` |
| Field affected | top-level AVB header flags |

Raw SHA-256:

```text
a30d4d8bc891320613f973153bf226fb1e580d9943fa1bc65025282584fbb6b5  /home/kali/pacextractor/pac_out/vbmeta-sign.img
6935f498ac76f8f175e64f3d51fa6db8aa398cdf2ac1d5b0a190ad2dbca74715  ../slot_a_ordered/00_vbmeta_a_live.img
```

The one-byte header hexdump difference is:

```diff
-00000070: 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
+00000070: 00 00 00 00 00 00 00 00 02 00 00 00 00 00 00 00
```

## D. AVB Metadata Comparison

| AVB metadata | Stock | Live | Match |
|---|---|---|---|
| Minimum libavb version | 1.0 | 1.0 | Yes |
| Authentication block size | 576 | 576 | Yes |
| Auxiliary block size | 17,472 | 17,472 | Yes |
| Algorithm | `SHA256_RSA4096` | `SHA256_RSA4096` | Yes |
| Public key SHA-1 | `7b595403a7796dc3c96d57d79583a21721893d20` | `7b595403a7796dc3c96d57d79583a21721893d20` | Yes |
| Rollback index | 0 | 0 | Yes |
| Rollback index location | 0 | 0 | Yes |
| Flags | `0` | `33554432` | No |
| Release string | `avbtool 1.3.0` | `avbtool 1.3.0` | Yes |
| Public key metadata | absent | absent | Yes |
| Chain descriptors | same set | same set | Yes |
| Hash descriptors | none listed | none listed | Yes |
| Hashtree descriptors | none listed | none listed | Yes |
| Exact AVB metadata match | No | No | No, flags differ |

Descriptor/chained partition coverage:

| Partition | Stock chain present | Live chain present | Metadata match | Rollback location | Public key SHA-1 |
|---|---:|---:|---:|---:|---|
| `boot` | yes | yes | yes | 1 | `98e711e004cbcad564b5692c88044726e92d631f` |
| `init_boot` | yes | yes | yes | 8 | `575d2803b0e6cfb05ff6175a97ccbe688c7260ff` |
| `vendor_boot` | yes | yes | yes | 18 | `8b41486347306814101259310ab4e64cc5bc7c5e` |
| `dtbo` | yes | yes | yes | 6 | `98e711e004cbcad564b5692c88044726e92d631f` |
| `vbmeta_system` | yes | yes | yes | 2 | `3ebd6c01eeac3fa4fcada01c69779fff5f631480` |
| `vbmeta_vendor` | yes | yes | yes | 4 | `d8aa47d8537098b37194d711215ca9db06e5f1ae` |
| `vbmeta_product` | yes | yes | yes | 5 | `06899cd3bfed63382fd3ca49bf53896fafe9f8ae` |
| `vbmeta_odm` | yes | yes | yes | 7 | `6042312d3d7129a4a92bab245f3993eb040c0766` |
| `vbmeta_system_ext` | yes | yes | yes | 3 | `ef800b08075bd3ec084ffd1ec44a710679875ecb` |
| `vendor_dlkm` | no | no | yes | n/a | n/a |
| `system_dlkm` | no | no | yes | n/a | n/a |
| `l_modem` | yes | yes | yes | 11 | `12e4192bdec0015a8427a318953c7b2d272830be` |
| `l_ldsp` | yes | yes | yes | 12 | `12e4192bdec0015a8427a318953c7b2d272830be` |
| `l_gdsp` | yes | yes | yes | 13 | `12e4192bdec0015a8427a318953c7b2d272830be` |
| `pm_sys` | yes | yes | yes | 14 | `12e4192bdec0015a8427a318953c7b2d272830be` |
| `l_agdsp` | yes | yes | yes | 15 | `12e4192bdec0015a8427a318953c7b2d272830be` |

No top-level chain descriptors are listed for `vendor_dlkm` or `system_dlkm`. The top-level image chains to `vbmeta_system_ext`, `vbmeta_vendor`, and related vbmeta partitions instead.

## E. Interpretation

Live `vbmeta_a` does not match stock exactly. It differs by one byte in the signed AVB header flags field.

There is evidence that live `vbmeta_a` was patched or modified: the top-level flags field changed from `0` to `33554432`, and `avbtool verify_image` reports a signature check failure for live `vbmeta_a`. Since the descriptors, keys, properties, rollback index, and release string all match stock, this looks like a minimal header flag mutation rather than a rebuilt or re-signed vbmeta image.

Under standard `avbtool`, stock `vbmeta-sign.img` has `Flags: 0`. Live `vbmeta_a` has `Flags: 33554432`, not the normal displayed values `1` or `2` used by `avbtool` for hashtree-disabled or verification-disabled flags. The practical result is still that the live vbmeta struct is no longer signature-valid.

This fits the LK strings:

```text
LOCK FLAG IS : UNLOCK
androidboot.verifiedbootstate=orange
device unlocked so not initializing digest
WARNNING: LOCK FLAG IS : UNLOCK, SKIP VERIFY
```

Because LK reports an unlocked/orange state and explicitly says it skips verification, the patched `init_boot_a` can boot even though top-level `vbmeta_a` is no longer signature-valid. The evidence points to verification being bypassed by unlocked LK state, not to a cleanly re-signed vbmeta chain.

## F. Clear Conclusion

Stock vs live `vbmeta`:
- Not byte-for-byte identical.
- Same size.
- Same descriptors, keys, algorithm, rollback index, release string, and build properties.
- Different top-level flags field.
- Live vbmeta signature check fails.

Current root chain:
- Best described as stock `boot_a`, stock `vendor_boot_a`, stock `dtbo_a`, patched `init_boot_a`, modified top-level `vbmeta_a`, and unlocked/orange LK behavior that skips AVB verification.

Boot-chain identity map:
- `vbmeta_a` supports the identity map because its chain descriptors for `boot`, `init_boot`, `vendor_boot`, `dtbo`, `vbmeta_system`, `vbmeta_vendor`, `vbmeta_product`, `vbmeta_odm`, `vbmeta_system_ext`, and Unisoc modem/DSP partitions match stock.
- It also explains why a patched `init_boot_a` can exist in the active path: the chain descriptor still points to `init_boot`, but unlocked/orange LK behavior prevents the modified payload from blocking boot.

## G. Next Recommended Step

Create a final `BOOT_CHAIN_FINDINGS.md` summary that consolidates:
- stock-identical `boot_a`
- Magisk-patched `init_boot_a`
- stock-identical `vendor_boot_a`
- stock-identical `dtbo_a`
- modified-but-descriptor-identical `vbmeta_a`
- LK unlocked/orange verification-skip behavior

Do not flash anything as part of this analysis step.
