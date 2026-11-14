# OPPO Pad (OPD2101) SukiSU Ultra 内核适配

本仓库仅包含 **适配补丁、构建产物与说明文档**，**不包含内核源码本体**。

## 源码来源与声明

内核源码来自第三方，非本人原创：

| 项目 | 信息 |
|---|---|
| 仓库 | https://gitee.com/univille/kernel_sm8250_oppo_opd2101 |
| 基线提交 | `f9c0f2a0994df98925663417c4692d3d1630b20a` |
| 原作者 | UniVille |

本仓库中的补丁是在上述源码基础上的**修改与新增**，遵循 GPL-2.0 协议分发。
SukiSU Ultra 内核部分代码来自 [SukiSU-Ultra/SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra)（GPL-2.0）。

## 设备信息

- 型号：OPPO Pad（OPD2101）
- SoC：SM8250 / kona（骁龙 870）
- 系统：Android 16 / ColorOS 16.0.1
- 内核：4.19.325-perf+（非 GKI）
- 内存：5.36 GB + 12 GB zram

## 当前状态

### ✅ 已验证可用（v2）

- SukiSU Ultra 内核集成
- 基础 Root / App Profile / 模块系统
- 原厂 vendor 模块放行（音频、充电、触控等）
- **镜像已实机验证可正常开机**

### ❌ 未完成

| 功能 | 状态 | 卡点 |
|---|---|---|
| SUSFS | 编译未通过 | 补丁已打完，卡在 `kernel/sched/core.c` 的 OPPO 定制代码与 `task_struct` 冲突 |
| KPM | 刷入卡第一屏 | KernelPatch 后端在 4.19 非 GKI 定制内核上启动失败，无日志可查 |

## 文件说明

```
patches/
  tracked-changes.patch   相对基线提交的完整 diff（含 SukiSU 集成、SUSFS 移植、编译修复）
  new-files-list.txt      28 个新增文件清单

newfiles/
  untracked.tar.gz        28 个新增文件实际内容（drivers/kernelsu/ 新结构）

build/
  Image                   内核镜像（与 v2 boot 内 kernel 完全一致）
  Module.symvers          符号表
  config                  完整内核配置

image/
  boot_sukisu_v2.img.gz   可开机的 boot 镜像（gzip 压缩，解压后 160MB）

docs/
  BUILD.md                复现构建步骤
  CHANGELOG.md            改动记录
```

## 关键配置

```
CONFIG_KSU=y
# CONFIG_KSU_FEATURE_ADBROOT is not set
# CONFIG_KSU_DEBUG is not set
CONFIG_KPM=y
# CONFIG_KSU_SUSFS is not set
```

## 校验值

| 文件 | SHA-256 |
|---|---|
| Image | `0d264f744965efbf8bafd4febe27b32fae0b7c176c434dd9bcb9a532d7e8ab70` |
| boot_sukisu_v2.img | `3ae48d38460f39bdd0026a582ee240390644ecfdb0ef8fd8efec5579465ebcca` |

> boot 镜像解压后 SHA-256 为 `3ae48d38...`，与实机 boot 分区一致，可确认是同一版本。

## 使用方式

### 复现编译

```bash
# 1. 克隆源码到基线提交
git clone https://gitee.com/univille/kernel_sm8250_oppo_opd2101.git kernel
cd kernel
git checkout f9c0f2a0994df98925663417c4692d3d1630b20a

# 2. 应用补丁
git apply ../patches/tracked-changes.patch

# 3. 解压新增文件
tar -xzf ../newfiles/untracked.tar.gz

# 4. 编译（详见 docs/BUILD.md）
```

### 刷入镜像

```bash
gunzip boot_sukisu_v2.img.gz
fastboot flash boot boot_sukisu_v2.img
fastboot reboot
```

**刷机前务必备份原厂 boot 分区。**

## 已知编译问题

从干净状态完整编译时会遇到以下问题（补丁中已修复）：

| 文件 | 问题 | 修复 |
|---|---|---|
| `arch/arm64/kernel/vdso32/sigreturn.S` | ARM 目标下宏不展开 | 补充 `__NR_compat_*` 兜底定义 |
| `mm/filemap.c` | 缺 `F2FS_SUPER_MAGIC` | 补 `#include <linux/magic.h>` |
| `mm/damon/core.c` | 缺 `get_task_struct` | 补 `#include <linux/sched/task.h>` |
| `kernel/rcu/tasks.h` | 5.x 代码混入，缺条件编译 | 用 `CONFIG_TASKS_RCU` 包裹 |
| `include/linux/filter.h` | 循环包含导致 `compat_uptr_t` 未定义 | 加 `CONFIG_COMPAT` 守卫 |
| `include/linux/sched.h` | SUSFS 在 `__GENKSYMS__` 分支误用 `ANDROID_KABI_USE` | 改回 `KABI_RESERVE` |

**注意**：本内核仓库原本就存在一些编译缺陷，原作者可能依赖预编译产物绕过。从零编译需逐个修复。

## 许可

- 内核代码：GPL-2.0
- 本仓库新增的适配代码：GPL-2.0

## 致谢

- [UniVille](https://gitee.com/univille) — 提供 OPD2101 内核源码
- [SukiSU-Ultra](https://github.com/SukiSU-Ultra/SukiSU-Ultra) — 内核级 Root 方案
- [simonpunk/susfs4ksu](https://gitlab.com/simonpunk/susfs4ksu) — SUSFS 隐藏方案（未完成）
