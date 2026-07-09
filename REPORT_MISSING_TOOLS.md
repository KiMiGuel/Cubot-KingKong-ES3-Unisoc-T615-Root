# Phase 1 Missing Tools Report

## Scope

This report covers only the tools requested for Phase 1 of the Cubot KingKong ES3 active slot A analysis.

No image files were modified, unpacked, patched, flashed, uploaded, or copied into git.

## Summary

Missing from `PATH`:

- `magiskboot` - intentionally deferred/ignored for now
- `mkdtimg` - optional DTBO helper missing; direct DTBO table parsing was used
- `dump_dtbo` - optional DTBO helper missing; direct DTBO table parsing was used

Present:

- `file`
- `hexdump`
- `xxd`
- `avbtool`
- `unpack_bootimg`
- `mkbootimg`
- `dtc`
- `lz4`
- `cpio`
- `gzip`

## Missing Tool Details

### avbtool

Status: present at `/usr/local/bin/avbtool`.

Impact: AVB/vbmeta descriptor inspection is now available. `avbtool info_image` was run against `../slot_a_ordered/00_vbmeta_a_live.img`, and the results were appended to `IMAGE_METADATA.txt`.

### magiskboot

Status: missing from `PATH`.

Impact: Magisk-specific boot image inspection cannot be performed with `magiskboot` unless the tool is installed later. Per current instruction, `magiskboot` is ignored/deferred for now and is not a Phase 1 blocker.

Local package-cache result:

```text
apt-cache search magiskboot
```

No matching package was returned by the local apt cache.

Exact upstream source retrieval command:

```bash
git clone --depth 1 https://github.com/topjohnwu/Magisk.git
```

Note: the local apt cache does not provide a direct `magiskboot` package. `magiskboot` is distributed with/buildable from Magisk upstream rather than as an available distro package in this environment.

### mkdtimg

Status: missing from `PATH`.

Impact: Android DTBO table dump/extract was not performed with `mkdtimg`. Phase 2 used direct DTBO table parsing and `dtc` decompile instead.

Local package-cache result:

```text
apt-cache search mkdtimg
```

No matching package was returned by the local apt cache.

Exact upstream source retrieval/install commands:

```bash
sudo apt install git python3
git clone https://android.googlesource.com/platform/system/libufdt
sudo install -m 0755 libufdt/utils/src/mkdtimg.py /usr/local/bin/mkdtimg
```

### dump_dtbo

Status: missing from `PATH`.

Impact: A separate `dump_dtbo` helper was not available. Phase 2 used direct DTBO table parsing and `dtc` decompile instead.

Local package-cache result:

```text
apt-cache search dump_dtbo
```

No matching package was returned by the local apt cache.

Suggested replacement:

```bash
mkdtimg dump ../slot_a_ordered/01_dtbo_a_live.img
```

Install `mkdtimg` using the commands in the `mkdtimg` section above.

## Installed Tool Evidence

```text
/usr/bin/file
/usr/bin/hexdump
/usr/bin/xxd
/usr/local/bin/avbtool
/usr/bin/unpack_bootimg
/usr/bin/mkbootimg
/usr/bin/dtc
/usr/bin/lz4
/usr/bin/cpio
/usr/bin/gzip
```

`command -v avbtool magiskboot` exited non-zero because `magiskboot` remains absent; `avbtool` was found at `/usr/local/bin/avbtool`.

Phase 2 also checked `mkdtimg` and `dump_dtbo`; both were absent from `PATH`.
