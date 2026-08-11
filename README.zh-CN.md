# 红米 K30S Ultra（apollon）SM8250 内核：开启 PID / IPC namespace

本 fork 在 **LineageOS `lineage-23.2`** 内核（红米 K30S Ultra / apollon，
M2007J3SC）上开启 **`CONFIG_PID_NS` / `CONFIG_IPC_NS`**，使 **DroidSpaces**
等 LXC 式容器运行时可以在该设备上运行。

[English](./README.md)

**补丁位置：** namespace 支持在
[`lineage-23.2-pidns`](https://github.com/kiiki290/android_kernel_xiaomi_sm8250/tree/lineage-23.2-pidns)
分支；默认分支（`lineage-23.2`）与上游 LineageOS 内核一致。

---

## 为什么需要这颗内核

- 官方 LineageOS 23.2 内核**禁用**了 PID namespace（defconfig 中
  `# CONFIG_PID_NS is not set`）；`CONFIG_IPC_NS` 同样关闭。
- **DroidSpaces**（以及其他 LXC 类运行时）**硬性要求** PID namespace：其
  requirements check 会失败，`unshare -p` / `unshare -i` 在官方内核上返回
  `EINVAL` —— 这是真正的内核级缺失，与 seccomp / SELinux 无关。
- 没有现成的第三方 apollon 内核开启 PID_NS，所以本 fork 自编一颗：**基线
  相同**（`4.19.325-cip131-st15-perf`）、**编译器相同**（官方 clang），只打开
  namespace 相关选项。

## 改动内容

相对 `lineage-23.2`：

| 文件 | 改动 |
|---|---|
| `arch/arm64/configs/vendor/kona-perf-pidns_defconfig` | 合并官方 apollon 配置（kona-perf + debugfs + xiaomi/sm8250-common + xiaomi/apollo）并加 `CONFIG_SYSVIPC=y`、`CONFIG_PID_NS=y`、`CONFIG_IPC_NS=y` |
| `.github/workflows/build-kernel.yml` | GitHub Actions 手动构建，使用官方 LineageOS 23.2 的 clang 工具链 |
| `drivers/clk/qcom/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `drivers/clk/qcom/mdss/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `techpack/display/pll/Makefile` | `ccflags-y += -I$(srctree)/$(src)`（加在已有 `ccflags-y` 块**内部**） |
| `techpack/camera-xiaomi-cas/drivers/cam_sensor_module/cam_cci/Makefile` | `ccflags-y += -I$(srctree)/$(src)` |
| `scripts/Makefile.build` | built-in.a / 多目标规则改用 `$(file ...)` + `ar @file`，使其低于内核 128 KB 单参数上限 |

> 配置 = 官方 apollon 合并配置 + 开启 `PID_NS` / `IPC_NS`（并依赖 `SYSVIPC`）。
> 其余改动是让这颗树能在官方 clang 21 下正常编译的构建修复（见下节）。

## 三个构建根因与修复

官方源码树在 LineageOS 自己的构建环境里编译。用 clang 在 GitHub Actions 上
复现时暴露出三个独立问题，均在本分支修复。

### 1. `trace.h` include 污染 → 隐式声明错误

**问题。** 全局 `-I` 使 `define_trace.h` 的 `#include "./trace.h"`（cwd 相对
路径，会回退到 `-I`）命中无关目录的 `trace.h`。`regmap.c` 在错误上下文展开了
`clk/qcom/trace.h` → “`check_trace_callback_type_clk_measure` 隐式声明”→ 被
`-Werror` 干掉。

**修复。** 不做全局 include 大锅饭：`KCFLAGS` 保持为空；所有 `-I` 由各目录的
`ccflags-y` 提供（GKI 模式）。官方 LineageOS 构建同样没有全局 `-I`。

### 2. 四个 Makefile 缺少自身目录的 `-I`

**问题。** `drivers/clk/qcom`、`drivers/clk/qcom/mdss`、
`techpack/display/pll`、`techpack/camera-xiaomi-cas/drivers/cam_sensor_module/cam_cci`
里的 `./trace.h`、`./pll_trace.h`、`cam_cci_dev.h` 报“No such file”。

**修复。** 给这 4 个 Makefile 各加 `ccflags-y += -I$(srctree)/$(src)`。（注意：
`techpack/display/pll/Makefile` 必须加在已有 `ccflags-y :=` 块**内部** —— 该
文件末尾有续行符反斜杠。）

### 3. `qcacld` built-in.a 触发 128 KB 单参数上限（`E2BIG`）

**问题。** GNU make 把**整条 recipe 作为单个 argv** 传给 `/bin/sh -c`。Wi-Fi
驱动（`qcacld`）built-in.a 把 **693 个对象路径列了两遍**（`update_lto_symversions`
的 for 循环 + `ar` 参数）≈ **110–130 KB**，超过内核 `MAX_ARG_STRLEN`
（**128 KB = 32 页**，单参数限制）。`execve` 报 `Argument list too long`
（`E2BIG`）/ `Error 127`。这**不是**总 `ARG_MAX`（4 MB）限制。

单靠 `printf | xargs`（上游 v5.19 修复 `cd968b97c492`）**不够**：它只把 `ar`
参数移出命令行，但 for 循环列表仍在 recipe 里 —— 两份 693 对象加起来还是超限。

**修复。** 用 **`$(file >$@.objs, <对象列表>)`**（GNU make ≥ 4.0 在 make 内部把
列表写入文件，完全不经过 argv），再让归档器用 **`ar @file`**（响应文件；需要
`llvm-ar`，`LLVM=1` 时提供）读回。

```make
cmd_ar_builtin = $(update_lto_symversions) \
	rm -f $@; \
	$(file >$@.objs,$(filter $(real-obj-y), $^)) \
	$(AR) rcSTP$(KBUILD_ARFLAGS) $@ @$@.objs; \
	rm -f $@.objs
```

---

## 构建

### 环境要求

- AOSP **clang r563880c**（clang 21.0.0）—— 与 LineageOS 23.2 官方使用的
  **同一套**工具链（workflow 引用了它的公开镜像）。
- `make` 与标准内核构建依赖。

### 手动构建

```bash
export PATH="$CLANG_BIN:$PATH"        # CLANG_BIN = 包含 clang 的目录
export ARCH=arm64 SUBARCH=arm64
make CC=clang LLVM=1 LLVM_IAS=1 vendor/kona-perf-pidns_defconfig
make CC=clang LLVM=1 LLVM_IAS=1 KCFLAGS="" -j$(nproc) Image
```

- 产物：`arch/arm64/boot/Image`（未压缩 arm64 内核，约 55 MB）。
- 结果：
  `Linux version 4.19.325-cip131-st15-perf-g33b52341d1ce ...`
  （`-g33b52341d1ce` = `lineage-23.2-pidns` tip 提交）。

### GitHub Actions 构建

`.github/workflows/build-kernel.yml` 在 `ubuntu-latest` 上构建同一内核。在
`lineage-23.2-pidns` 分支上运行 **workflow_dispatch**（GitHub → Actions →
*build-kernel-pidns* → *Run workflow*）。产物：`Image`、`.config`、`build.log`。

## 刷入

> ⚠️ 需要**已解锁 bootloader** 和 **root**（`adb root` 或 Magisk）。刷机有风险，
> 请先备份。

1. **用自定义内核重新打包官方 boot 镜像**（在手机上，或任何有 `magiskboot`
   的环境）：

   ```bash
   magiskboot unpack boot.img     # 解出 kernel / ramdisk / dtb / header
   # 把解出的 `kernel` 换成我们的 `Image`
   cp Image kernel
   magiskboot repack boot.img     # -> new-boot.img
   ```

2. **用 Magisk 修补** —— *Magisk app → 安装 → 修补一个 boot 镜像文件* → 选择
   打包好的镜像 → 得到 `magisk_patched.img`。

3. **把修补镜像刷到 boot 分区**：

   ```bash
   adb push magisk_patched.img /sdcard/Download/
   adb root
   adb shell "dd if=/sdcard/Download/magisk_patched.img of=/dev/block/bootdevice/by-name/boot bs=4M"
   adb reboot
   ```

4. **验证**（见下节）。

> **LineageOS OTA 之后内核与 root 都会被覆盖** —— 需重走此流程（重新修补新版
> 官方 boot 镜像）。

## 验证

```bash
# 当前运行的内核是我们的
adb shell cat /proc/version
#   Linux version 4.19.325-cip131-st15-perf-g33b52341d1ce ...

# PID namespace 现已可用（官方内核是 EINVAL）
adb shell su -c 'unshare -p /system/bin/true'; echo $?
# 0

# IPC namespace 现已可用
adb shell su -c 'unshare -i /system/bin/true'; echo $?
# 0
```

DroidSpaces 自身的 requirements check 报告 PID / IPC namespace 已存在。

## 下载

预编译内核：见
[Releases](https://github.com/kiiki290/android_kernel_xiaomi_sm8250/releases)
页面 —— `Image` 产物及其 SHA-256。**只发布内核**；不发布 boot.img /
Magisk 修补文件。

## 许可

[LineageOS/android_kernel_xiaomi_sm8250](https://github.com/LineageOS/android_kernel_xiaomi_sm8250)
的 fork，以 **GPL-2.0** 发布。本 fork 的所有改动同样适用该许可。

**免责声明：** 本内核按原样提供，不提供任何担保。刷写自定义内核可能使保修失效
或造成数据丢失，风险自负。仅适用于红米 K30S Ultra / apollon。
