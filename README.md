# Xiaomi SM8250 kernel with PID / IPC namespace support

> This fork enables `CONFIG_PID_NS` and `CONFIG_IPC_NS` on the **LineageOS
> `lineage-23.2`** kernel for the **Redmi K30S Ultra (apollon, M2007J3SC)**,
> so that LXC-style container runtimes such as **DroidSpaces** can run on the
> device.
>
> 本 fork 在 LineageOS 23.2 内核上开启 **PID / IPC namespace**，使红米
> K30S Ultra (apollon) 可以运行 DroidSpaces 等 LXC 容器。

**Where the patches are:** the namespace support lives on the
[`lineage-23.2-pidns`](https://github.com/kiiki290/android_kernel_xiaomi_sm8250/tree/lineage-23.2-pidns)
branch. The default branch (`lineage-23.2`) tracks the stock upstream
LineageOS kernel.

> 补丁在 `lineage-23.2-pidns` 分支；默认分支与上游 LineageOS 内核一致。

---

## Why / 为什么需要这颗内核

- The **stock** LineageOS 23.2 kernel **disables** PID namespaces
  (`# CONFIG_PID_NS is not set` in the defconfig); `CONFIG_IPC_NS` is also off.
- **DroidSpaces** (and other LXC-style runtimes) **require** a PID namespace:
  its requirement check fails, and `unshare -p` / `unshare -i` return `EINVAL`
  on the stock kernel — a real kernel-level absence, not seccomp or SELinux.
- No readily available third-party kernel for apollon enables PID_NS, so this
  fork builds one: same base (`4.19.325-cip131-st15-perf`), same compiler
  (official clang), only the namespace options turned on.

---

## What changed / 改动内容

Relative to `lineage-23.2`:

| File | Change |
|---|---|
| `arch/arm64/configs/vendor/kona-perf-pidns_defconfig` | Merged apollon config matching the official build (kona-perf + debugfs + xiaomi/sm8250-common + xiaomi/apollo) plus `CONFIG_SYSVIPC=y`, `CONFIG_PID_NS=y`, `CONFIG_IPC_NS=y` |
| `.github/workflows/build-kernel.yml` | GitHub Actions build (manual dispatch) using the official LineageOS 23.2 clang toolchain |
| `drivers/clk/qcom/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `drivers/clk/qcom/mdss/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `techpack/display/pll/Makefile` | `ccflags-y += -I$(srctree)/$(src)` (added *inside* the existing `ccflags-y` block) |
| `techpack/camera-xiaomi-cas/drivers/cam_sensor_module/cam_cci/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `scripts/Makefile.build` | Built-in.a / multi-obj rules use `$(file ...)` + `ar @file` to stay under the kernel's 128 KB single-argument limit |

> 配置 = 官方 apollon 合并配置 + 开启 `PID_NS` / `IPC_NS`（并依赖 `SYSVIPC`）。
> 其余改动是让这颗树能在官方 clang 21 下正常编译的构建修复（见下节）。

---

## The three build issues fixed / 三个构建根因与修复

The official tree builds inside LineageOS's own build environment. Reproducing
it with clang on GitHub Actions surfaced three independent problems, all fixed
in this branch.

### 1. `trace.h` include pollution → implicit-declaration error

**Problem.** A global `-I` flag made `define_trace.h`'s `#include "./trace.h"`
(a cwd-relative fallback that resolves through `-I`) pick up an unrelated
directory's `trace.h`. `regmap.c` ended up expanding `clk/qcom/trace.h` in the
wrong context → “implicit declaration of `check_trace_callback_type_clk_measure`”
→ killed by `-Werror`.

**Fix.** No global include soup: `KCFLAGS` stays empty; every `-I` is provided
per-directory via `ccflags-y` (the GKI pattern). The stock LineageOS build uses
no global `-I` either.

### 2. Missing per-directory `-I` in four Makefiles

**Problem.** `./trace.h`, `./pll_trace.h`, `cam_cci_dev.h` were “No such file”
in `drivers/clk/qcom`, `drivers/clk/qcom/mdss`, `techpack/display/pll` and
`techpack/camera-xiaomi-cas/drivers/cam_sensor_module/cam_cci`.

**Fix.** Add `ccflags-y += -I$(srctree)/$(src)` to each of those four Makefiles.
(Note: in `techpack/display/pll/Makefile` the line must go *inside* the existing
`ccflags-y :=` block — the file ends with a line-continuation backslash.)

### 3. `qcacld` built-in.a hits the 128 KB single-argument limit (`E2BIG`)

**Problem.** GNU make passes the **entire recipe as a single argv** to
`/bin/sh -c`. The Wi-Fi driver (`qcacld`) built-in.a lists **693 object paths
twice** (in the `update_lto_symversions` for-loop and in the `ar` arguments) ≈
**110–130 KB**, above the kernel's `MAX_ARG_STRLEN` (**128 KB = 32 pages**,
per-argument). `execve` fails with `Argument list too long` (`E2BIG`) /
`Error 127`. This is *not* the total `ARG_MAX` (4 MB) limit.

`printf | xargs` alone (the upstream v5.19 fix `cd968b97c492`) is **not
enough**: it moves the `ar` arguments out of the command, but the for-loop list
stays in the recipe — both copies together still exceed the limit.

**Fix.** Use **`$(file >$@.objs, <obj list>)`** (GNU make ≥ 4.0 writes the list
to a file internally, never through argv) and feed it back to the archiver with
**`ar @file`** (a response file; requires `llvm-ar`, provided by `LLVM=1`).

```make
cmd_ar_builtin = $(update_lto_symversions) \
	rm -f $@; \
	$(file >$@.objs,$(filter $(real-obj-y), $^)) \
	$(AR) rcSTP$(KBUILD_ARFLAGS) $@ @$@.objs; \
	rm -f $@.objs
```

---

## Build / 构建

### Requirements / 环境要求

- AOSP **clang r563880c** (clang 21.0.0) — the *same* toolchain LineageOS 23.2
  uses officially (the workflow references a public mirror of it).
- `make` and the standard kernel build dependencies.

### Manually

```bash
export PATH="$CLANG_BIN:$PATH"        # CLANG_BIN = directory containing clang
export ARCH=arm64 SUBARCH=arm64
make CC=clang LLVM=1 LLVM_IAS=1 vendor/kona-perf-pidns_defconfig
make CC=clang LLVM=1 LLVM_IAS=1 KCFLAGS="" -j$(nproc) Image
```

- Output: `arch/arm64/boot/Image` (uncompressed arm64 kernel, ~55 MB).
- The result reports:
  `Linux version 4.19.325-cip131-st15-perf-g33b52341d1ce ...`
  (`-g33b52341d1ce` = the `lineage-23.2-pidns` tip commit).

### With GitHub Actions

`.github/workflows/build-kernel.yml` builds the same kernel on `ubuntu-latest`.
Run **workflow_dispatch** on the `lineage-23.2-pidns` branch
(GitHub → Actions → *build-kernel-pidns* → *Run workflow*). Artifacts:
`Image`, `.config`, `build.log`.

---

## Flash / 刷入

> ⚠️ Requires an **unlocked bootloader** and **root** (`adb root` or Magisk).
> Flashing is destructive — back up everything first.
> 需要已解锁 bootloader；刷机有风险，请先备份。

1. **Repack the official boot image with the custom kernel** (on the phone,
   or in a `magiskboot`-capable environment):

   ```bash
   magiskboot unpack boot.img     # extracts kernel / ramdisk / dtb / header
   # replace the extracted `kernel` file with our `Image`
   cp Image kernel
   magiskboot repack boot.img     # -> new-boot.img
   ```

2. **Patch with Magisk** — *Magisk app → Install → Patch a boot image file* →
   select the repacked image → produces `magisk_patched.img`.

3. **Flash the patched image to the boot partition**:

   ```bash
   adb push magisk_patched.img /sdcard/Download/
   adb root
   adb shell "dd if=/sdcard/Download/magisk_patched.img of=/dev/block/bootdevice/by-name/boot bs=4M"
   adb reboot
   ```

4. **Verify** (see below).

> After a **LineageOS OTA**, both the kernel **and** root are overwritten —
> re-apply this procedure (re-patch the new official boot image).
> LineageOS OTA 之后内核与 root 都会被覆盖，需要重走一遍此流程。

---

## Verification / 验证

```bash
# The running kernel is ours
adb shell cat /proc/version
#   Linux version 4.19.325-cip131-st15-perf-g33b52341d1ce ...

# PID namespace now works (was EINVAL on stock)
adb shell su -c 'unshare -p /system/bin/true'; echo $?
# 0

# IPC namespace now works
adb shell su -c 'unshare -i /system/bin/true'; echo $?
# 0
```

DroidSpaces' own requirements check reports the PID / IPC namespace feature as
present.

---

## Downloads / 下载

Prebuilt kernel: see the
[Releases](https://github.com/kiiki290/android_kernel_xiaomi_sm8250/releases)
page — the `Image` artifact plus its SHA-256. **Only the kernel is published**;
no boot images and no Magisk-patched files.

> 只发布内核 `Image` 与其 SHA-256；不发布 boot.img / Magisk 修补文件。

---

## License / 许可

A fork of
[LineageOS/android_kernel_xiaomi_sm8250](https://github.com/LineageOS/android_kernel_xiaomi_sm8250),
released under **GPL-2.0**. All changes in this fork carry the same license.

**Disclaimer:** this kernel is provided as-is, without warranty. Flashing a
custom kernel may void your warranty or cause data loss; you do this at your
own risk. It targets the Redmi K30S Ultra / apollon only.
