# 构建说明

## 环境

本次构建使用：

- 平台：ARM64（本机原生编译，非交叉编译）
- 系统：Alpine Linux 3.24
- 编译器：Clang 22.1.3 + LLD 22.1.3
- 依赖：`clang22 lld22 llvm22 flex bison elfutils-dev openssl-dev perl bc cpio lz4 zstd binutils binutils-arm-none-eabi linux-headers zlib-dev zlib-static`

原作者使用的是 ZyC Clang 23.0.0git，本仓库使用 Alpine 自带的 Clang 22.1.3，配置需做相应调整。

## 步骤

```bash
# 1. 准备源码
git clone https://gitee.com/univille/kernel_sm8250_oppo_opd2101.git kernel
cd kernel
git checkout f9c0f2a0994df98925663417c4692d3d1630b20a

# 2. 应用补丁
git apply /path/to/tracked-changes.patch

# 3. 解压新增文件
tar -xzf /path/to/untracked.tar.gz

# 4. 准备配置
# 从设备上提取的原始配置为基准，或使用本仓库 build/config
mkdir out
cp /path/to/config out/.config

# 5. 生成配置
make O=out ARCH=arm64 olddefconfig

# 6. 编译
export PATH=/usr/lib/llvm22/bin:$PATH
make -C . O=out ARCH=arm64 \
  CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm \
  OBJCOPY=llvm-objcopy OBJDUMP=llvm-objdump STRIP=llvm-strip \
  CLANG_TRIPLE=aarch64-linux-gnu- \
  CROSS_COMPILE=aarch64-linux-gnu- \
  CROSS_COMPILE_ARM32=arm-none-eabi- \
  -j2 Image
```

产物：`out/arch/arm64/boot/Image`

## 已知问题

### 1. vdso32 汇编宏不展开

`arch/arm64/kernel/vdso32/sigreturn.S` 在 clang 的 ARM 目标下，`#include <asm/asm-offsets.h>` 的宏不生效。补丁中已加入兜底定义。

### 2. 需要从干净状态编译

**不要复用旧的 `out/` 目录。** 增量编译可能跳过有问题的文件，导致"看起来能编过"。清理后完整编译才能暴露真实问题。

### 3. OPPO 定制代码冲突

`kernel/sched/core.c` 在 `OPLUS_FEATURE_SCHED_ASSIST` 分支中使用 `p->last_wake_ts`、`p->ravg`，这些字段只在 `CONFIG_SCHED_WALT=y` 时定义。

当前状态：**此问题未解决**，SUSFS 编译卡在这里。

## 打包成 boot 镜像

内核 `Image` 需要包装进 boot 镜像。本设备为 Android boot header v2：

```python
# 关键参数
page_size = 4096
header_version = 2
# 需要保留原 boot 的 ramdisk、DTB、cmdline、AVB footer
```

注意：
- 内核未压缩（`CONFIG_BUILD_ARM64_UNCOMPRESSED_KERNEL=y`），不要加 gzip
- AVB footer 需同步更新 `image_size` 和 `vbmeta_offset` 字段
- `header id[]` 字段（偏移 576，20 字节）保留了原厂值，非 kernel 哈希

参考实现见本次适配过程，或使用 `magiskboot` / `kptools` 等工具。
