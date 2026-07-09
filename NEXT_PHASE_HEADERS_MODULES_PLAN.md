# Next Phase: Android 5.15 Live Headers and Module Build Environment

This plan is for building and validating kernel modules against the currently running stock Cubot KingKong ES3 kernel. It is not a flashing, boot image patching, or kernel replacement plan.

Target kernel:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115
```

Live sources expected on the phone:

```text
/proc/config.gz
/sys/kernel/kheaders.tar.xz
```

## Goals

Primary goal:

- Build or validate at least one simple external module against the live stock kernel environment.

Secondary goals:

- Determine whether the live headers are sufficient for module builds.
- Determine whether `CONFIG_MODVERSIONS` requires exact `Module.symvers` or matching symbol CRC data.
- Establish a repeatable local module-build workspace.
- Evaluate whether external USB Wi-Fi, Bluetooth, or HID-related modules are realistic on this kernel.

Non-goals:

- Do not flash images.
- Do not repack boot images.
- Do not replace the stock kernel.
- Do not assume a module can load just because it builds.
- Do not claim USB Wi-Fi or Bluetooth support is solved until a module builds, loads, and hardware/firmware are validated.

## Prerequisites

Host tools:

```text
adb
make
bc
bison
flex
openssl
libssl-dev
libelf-dev
perl
python3
clang or Android-compatible cross compiler
llvm tools if the live kernel was built with LLVM
```

Phone requirements:

```text
root shell access
read access to /proc/config.gz
read access to /sys/kernel/kheaders.tar.xz
uname -r access
dmesg access if possible
lsmod or /proc/modules access
modinfo/insmod/rmmod if available
```

Recommended staging directory on host:

```text
kernel_live_headers/
```

Keep copied phone evidence as text/config/header artifacts only. Do not copy raw firmware images into the repo.

## Evidence to Collect From the Phone

Collect these first:

```sh
adb shell su -c 'uname -a'
adb shell su -c 'uname -r'
adb shell su -c 'zcat /proc/config.gz' > live_kernel.config
adb shell su -c 'cat /proc/version'
adb shell su -c 'cat /proc/modules' > live_proc_modules.txt
adb shell su -c 'ls -l /sys/kernel/kheaders.tar.xz'
adb shell su -c 'cat /sys/kernel/kheaders.tar.xz' > kheaders.tar.xz
```

If available:

```sh
adb shell su -c 'cat /proc/kallsyms' > live_kallsyms.txt
adb shell su -c 'find /vendor/lib/modules /vendor_dlkm/lib/modules /odm/lib/modules /odm_dlkm/lib/modules -type f -name "*.ko" 2>/dev/null'
adb shell su -c 'find / -name Module.symvers -o -name modules.order 2>/dev/null'
adb shell su -c 'dmesg | tail -300' > live_dmesg_tail.txt
```

Important values to record:

| Evidence | Reason |
|---|---|
| `uname -r` | Must match module vermagic expectations |
| `/proc/config.gz` | Source of live kernel config |
| `/sys/kernel/kheaders.tar.xz` | Kernel header tree exposed by the running kernel |
| `CONFIG_MODVERSIONS` | Determines whether symbol CRCs matter |
| `CONFIG_MODULES` | Must be enabled for loadable modules |
| `CONFIG_MODULE_UNLOAD` | Determines whether test modules can be unloaded |
| `CONFIG_KALLSYMS` | Helps inspect exported symbols |
| `Module.symvers` | Provides symbol CRCs when modversions are enabled |
| existing `.ko` vermagic | Shows ABI expectations of stock modules |

## Stage kheaders

Create a clean host workspace:

```sh
mkdir -p kernel_live_headers
cd kernel_live_headers
mkdir -p kheaders
tar -xJf ../kheaders.tar.xz -C kheaders
```

Then inspect the extracted tree:

```sh
find kheaders -maxdepth 3 -type f | sort | sed -n '1,120p'
find kheaders -name Makefile -o -name Kconfig -o -name Module.symvers -o -name modules.order
```

Expected outcomes:

- A usable kernel header tree should include enough generated headers and Makefiles for external module builds.
- It may not include full kernel source.
- It may not include `Module.symvers`.

## Prepare `.config`

Create `.config` from the live device:

```sh
zcat live_kernel.config.gz > .config
```

If the file was already collected as plain text:

```sh
cp live_kernel.config .config
```

Place `.config` at the root of the staged header tree expected by the kernel build system.

Check key config values:

```sh
grep -E '^(CONFIG_MODULES|CONFIG_MODULE_UNLOAD|CONFIG_MODVERSIONS|CONFIG_KALLSYMS|CONFIG_KALLSYMS_ALL|CONFIG_IKHEADERS|CONFIG_LOCALVERSION)=' .config
```

Record:

| Config | Meaning |
|---|---|
| `CONFIG_MODULES=y` | Required to load external modules |
| `CONFIG_MODULE_UNLOAD=y` | Allows clean unload tests |
| `CONFIG_MODVERSIONS=y` | Requires matching exported symbol CRCs |
| `CONFIG_IKHEADERS=y` or module | Explains `/sys/kernel/kheaders.tar.xz` |
| `CONFIG_LOCALVERSION` | Can affect vermagic |

## Prepare the Header Tree

Try the least invasive preparation first:

```sh
make ARCH=arm64 olddefconfig
make ARCH=arm64 modules_prepare
```

If the kernel was built with LLVM, use:

```sh
make ARCH=arm64 LLVM=1 olddefconfig
make ARCH=arm64 LLVM=1 modules_prepare
```

If the tree is only kheaders and lacks targets required by `modules_prepare`, document the exact missing file or target. Do not invent generated headers by hand unless the missing input is clearly recoverable from the live kheaders archive.

## Check vermagic

Build a test module and inspect vermagic before trying to load it:

```sh
modinfo hello.ko
```

Expected vermagic should be close to:

```text
5.15.178-android13-8-00012-g4ea0fcb5d130-ab13530115 SMP preempt mod_unload modversions aarch64
```

Compare against stock vendor modules extracted from `vendor_boot_a`, which showed:

```text
5.15.178-android13-8-g5b7f481f7f44-dirty SMP preempt mod_unload modversions aarch64
```

This mismatch must be handled carefully. Existing vendor modules may have a different embedded vermagic string while still working through the vendor boot environment. The live `uname -r`, generated module vermagic, and actual insertion behavior are the deciding evidence.

## Detect and Handle CONFIG_MODVERSIONS

Check:

```sh
grep '^CONFIG_MODVERSIONS=' .config
```

If `CONFIG_MODVERSIONS` is not enabled:

- vermagic and exported symbol availability are still important.
- Missing symbol CRCs are less likely to block insertion.

If `CONFIG_MODVERSIONS=y`:

- `Module.symvers` becomes important.
- A module may build but fail to load with symbol version errors.
- `modules_prepare` alone may not generate correct CRCs.

Search for symbol data:

```sh
find . -name Module.symvers -o -name '*.symvers'
grep -R 'module_layout' . 2>/dev/null | head
```

If no usable `Module.symvers` exists:

- document that external modules may fail at load time
- try a minimal module anyway to capture exact errors
- investigate whether symbol CRCs can be recovered from stock modules, build artifacts, or vendor source drops

## Hello-World Module Test Plan

Create `hello.c`:

```c
#include <linux/init.h>
#include <linux/module.h>

static int __init hello_init(void)
{
    pr_info("es3_hello: loaded\n");
    return 0;
}

static void __exit hello_exit(void)
{
    pr_info("es3_hello: unloaded\n");
}

module_init(hello_init);
module_exit(hello_exit);

MODULE_LICENSE("GPL");
MODULE_DESCRIPTION("Cubot ES3 live-header module test");
```

Create `Makefile`:

```make
obj-m += hello.o
KDIR ?= /path/to/kernel_live_headers/kheaders

all:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=arm64 modules

clean:
	$(MAKE) -C $(KDIR) M=$(PWD) ARCH=arm64 clean
```

Build:

```sh
make
modinfo hello.ko
```

Copy and test only after build and vermagic inspection:

```sh
adb push hello.ko /data/local/tmp/
adb shell su -c 'insmod /data/local/tmp/hello.ko'
adb shell su -c 'dmesg | tail -80'
adb shell su -c 'rmmod hello'
adb shell su -c 'dmesg | tail -80'
```

If unloading is disabled, skip `rmmod` and document that test modules may require reboot to clear.

## Module ABI Validation Plan

Validate in layers:

1. Build a minimal module with no nontrivial external dependencies.
2. Confirm `modinfo` vermagic.
3. Try insertion on the rooted phone.
4. Record exact `dmesg` output.
5. If it loads, test unload if supported.
6. If it fails, classify the failure before changing anything.

Common failure classes:

| Error | Likely meaning |
|---|---|
| `invalid module format` | vermagic, architecture, signature, or ABI mismatch |
| `disagrees about version of symbol module_layout` | `CONFIG_MODVERSIONS` CRC mismatch |
| `disagrees about version of symbol <name>` | Missing or mismatched `Module.symvers` CRC |
| `Unknown symbol <name>` | Required exported symbol unavailable |
| `Exec format error` | Architecture or module format mismatch |
| `Operation not permitted` | SELinux, lockdown, module loading policy, or root context problem |
| module loads but device fails | Driver logic, firmware, hardware, or DT dependency issue |

## USB Wi-Fi, Bluetooth, and HID Feasibility Checks

Before building device drivers, collect capability evidence:

```sh
adb shell su -c 'zcat /proc/config.gz | grep -E "CONFIG_USB|CONFIG_CFG80211|CONFIG_MAC80211|CONFIG_BT|CONFIG_HID|CONFIG_INPUT"'
adb shell su -c 'ls /sys/bus/usb/devices'
adb shell su -c 'dmesg | grep -iE "usb|otg|typec|xhci|dwc|wifi|wlan|bluetooth|hid" | tail -200'
adb shell su -c 'lsmod'
```

For USB Wi-Fi:

- Check `CONFIG_CFG80211`.
- Check `CONFIG_MAC80211`.
- Check USB host/OTG support.
- Check firmware search paths.
- Confirm the exact chipset before choosing a driver.

For Bluetooth USB adapters:

- Check `CONFIG_BT`.
- Check `CONFIG_BT_HCIUSB` or equivalent support.
- Confirm firmware requirements.
- Confirm userspace stack availability.

For HID:

- Check `CONFIG_HID`.
- Check USB HID support.
- Test with simple HID devices before specialized devices.

Do not assume NetHunter features imply kernel driver availability. NetHunter userspace can be present while required kernel modules are missing or impossible to load under current ABI constraints.

## Handling Missing Module.symvers

If `Module.symvers` is missing and `CONFIG_MODVERSIONS=y`, options are:

1. Search the live phone and extracted stock firmware for a matching `Module.symvers`.
2. Search vendor kernel source or build artifacts for the exact kernel build.
3. Try to recover symbol CRC expectations from existing stock modules for limited diagnostics.
4. Build minimal modules and record exact loader errors.
5. Stop before attempting invasive work if symbol CRC mismatch blocks even hello-world module insertion.

Do not disable `CONFIG_MODVERSIONS` locally and assume that solves loadability. The running kernel decides the ABI rules.

## Stop Conditions

Stop the next phase if any of these occur:

- `uname -r` does not match the expected target and the cause is not understood.
- `/proc/config.gz` cannot be collected.
- `/sys/kernel/kheaders.tar.xz` cannot be collected or unpacked.
- The kheaders tree cannot run external module builds and the missing pieces are not recoverable.
- `CONFIG_MODULES` is not enabled.
- A minimal module builds but fails to load due to unresolved ABI or symbol CRC mismatch that cannot be resolved.
- Any test causes instability, boot loops, or repeated kernel panics.
- The work would require flashing, boot image repacking, or replacing the active kernel.

## Deliverables for the Next Phase

Suggested outputs:

```text
LIVE_KERNEL_HEADER_EVIDENCE.md
MODULE_BUILD_ENVIRONMENT.md
HELLO_MODULE_TEST_RESULTS.md
USB_MODULE_FEASIBILITY.md
```

Each result should distinguish:

- collected evidence
- commands run
- build result
- load result
- dmesg result
- remaining blocker
