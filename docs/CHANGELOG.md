# 改动记录

基线：`f9c0f2a0994df98925663417c4692d3d1630b20a`（UniVille kernel_sm8250_oppo_opd2101）

## 一、SukiSU Ultra 集成

### 移除旧 KernelSU 文件（20 个）

```
drivers/kernelsu/allowlist.c/h
drivers/kernelsu/apk_sign.c/h
drivers/kernelsu/arch.h
drivers/kernelsu/core_hook.c/h
drivers/kernelsu/embed_ksud.c
drivers/kernelsu/export_symbol.txt
drivers/kernelsu/include/ksu_hook.h
drivers/kernelsu/klog.h
drivers/kernelsu/ksu.c/h
drivers/kernelsu/ksud.c/h
drivers/kernelsu/manager.h
drivers/kernelsu/sucompat.c
drivers/kernelsu/throne_tracker.c/h
```

### 新增 SukiSU 结构（28 个新文件）

```
drivers/kernelsu/ksu_main.c
drivers/kernelsu/kernel_includes.h
drivers/kernelsu/extension/        (KPM、动态管理等扩展)
drivers/kernelsu/feature/          (sucompat、adb_root 等)
drivers/kernelsu/hook/             (hook 实现)
drivers/kernelsu/include/          (公共头文件)
drivers/kernelsu/infra/            (基础设施)
drivers/kernelsu/manager/          (管理器交互)
drivers/kernelsu/policy/           (App Profile 策略)
drivers/kernelsu/preinit/
drivers/kernelsu/runtime/          (ksud 交互)
drivers/kernelsu/sulog/
drivers/kernelsu/supercall/        (supercall 分发)
include/linux/ksu_hook.h
```

### 内核 hook 接入

修改以下文件注入 hook 调用：

| 文件 | 用途 |
|---|---|
| `fs/exec.c` | execve hook（su 提权、ksud 启动） |
| `fs/open.c` | faccessat hook |
| `fs/stat.c` | stat/fstat hook（Android 16 需要） |
| `fs/read_write.c` | vfs_read hook（ksud init.rc 注入） |
| `fs/namei.c` | 路径相关 |
| `kernel/reboot.c` | sys_reboot supercall |
| `kernel/sys.c` | setresuid hook |

## 二、原厂模块兼容

`kernel/module.c` 两处修改：

```c
// 1. 符号版本校验放行
static int check_version(...) {
    ...
    if (versions[i].crc == crcval)
        return 1;
    // 原: goto bad_version;  → 改为警告后放行
    pr_warn("%s: disagrees about version of symbol %s\n", info->name, symname);
    return 1;
}

// 2. 模块签名校验放行
static int module_sig_check(struct load_info *info, int flags) {
    info->sig_ok = false;
    return 0;   // 直接放行
}
```

**原因**：`/vendor/lib/modules` 下 41 个原厂模块的符号 CRC 与 vmlinux 不同（它们是为原厂内核编译的），强制校验会导致音频、充电、触控等模块无法加载。

**注意**：`CONFIG_MODULE_SIG_FORCE=y` 仍保留在配置中，只是策略上放行。

## 三、SUSFS 移植（未完成）

采用 SUSFS v1.5.5 官方 4.19 补丁（`simonpunk/susfs4ksu` 分支 `kernel-4.19`）。

### 改动的文件

```
fs/Makefile              加入 susfs.o
fs/dcache.c              sus_path 钩子
fs/namei.c               sus_path 钩子（17 个 hunk）
fs/namespace.c           挂载相关
fs/notify/fdinfo.c       挂载信息隐藏
fs/overlayfs/inode.c     overlayfs 支持
fs/overlayfs/readdir.c
fs/overlayfs/super.c
fs/proc/cmdline.c        cmdline 伪装
fs/proc/fd.c
fs/proc/task_mmu.c       sus_map
fs/proc_namespace.c
fs/readdir.c
fs/stat.c                sus_kstat
fs/statfs.c
kernel/kallsyms.c        符号隐藏
kernel/sys.c             uname 伪装
include/linux/mount.h    增加 susfs_mnt_id_backup 字段
include/linux/sched.h    增加 susfs_task_state 字段
```

### 新增

```
fs/susfs.c
include/linux/susfs.h
include/linux/susfs_def.h
include/linux/sus_su.h
```

### 未完成原因

`kernel/sched/core.c` 在 `OPLUS_FEATURE_SCHED_ASSIST` 分支使用 `p->last_wake_ts`、`p->ravg`，
这些字段定义在 `#ifdef CONFIG_SCHED_WALT` 块内。编译时宏未生效，导致编译失败。

配置中 `CONFIG_SCHED_WALT=y` 已开启，但预处理阶段未传入，根因待查。

## 四、编译问题修复

从干净状态编译时暴露的问题（原作者可能依赖增量产物绕过）：

| 文件 | 问题 |
|---|---|
| `arch/arm64/include/asm/vdso/compat_gettimeofday.h` | 补 `__NR_compat_*` 兜底宏 |
| `arch/arm64/kernel/vdso32/sigreturn.S` | 补 `COMPAT_SIGFRAME_REGS_OFFSET` 等常量 |
| `mm/filemap.c` | 补 `#include <linux/magic.h>` |
| `mm/damon/core.c` | 补 `#include <linux/sched/task.h>` |
| `kernel/rcu/tasks.h` | 5.x 代码用 `CONFIG_TASKS_RCU` 包裹 |
| `include/linux/filter.h` | `compat_sock_fprog` 加 `CONFIG_COMPAT` 守卫 |
| `include/linux/sched.h` | `__GENKSYMS__` 分支改回 `ANDROID_KABI_RESERVE` |

## 五、KPM 相关（未完成）

- 内核启用 `CONFIG_KPM=y`
- 后端 `SukiSU_KernelPatch_patch` 已编译出 `kpimg`
- 打入 boot 后卡第一屏，无日志（内核未到日志初始化阶段）
- 暂停

## 配置变更

相对原厂配置：

```
CONFIG_KSU=y                    # 新增
CONFIG_KPM=y                    # 新增
CONFIG_KALLSYMS_ALL=y           # 已有
# CONFIG_KSU_FEATURE_ADBROOT is not set
# CONFIG_KSU_DEBUG is not set
# CONFIG_KSU_SUSFS is not set    # 暂时关闭（编译未通过）
```

其余保持原厂配置不变。
