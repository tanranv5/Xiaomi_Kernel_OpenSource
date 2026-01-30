# KernelSU/SukiSU 模块安装问题解决方案

## 问题分析

### 核心问题
- **现象**：开启 vfs_read hook 会卡 MI（小米 Logo）；禁用 vfs_read hook 能开机但模块无法安装
- **根因**：ksud 未生效/未注入

### 为什么必须搞 vfs_read hook？

**vfs_read hook 的核心作用**：在 init 进程读取 `/system/etc/init/hw/init.rc` 时，动态注入 KernelSU 的 RC 脚本内容。

没有这个注入：
1. `ksud` 守护进程不会在 `post-fs-data` 阶段启动
2. 模块的脚本不会执行
3. 模块挂载不会发生
4. **结果**：模块"安装"了但实际不生效

---

## 方案对比

### 方案一：优化 kprobe 方式的 vfs_read hook（推荐用于无法修改内核源码时）

**优化策略**：
- 仅在 init 进程（pid=1）且 comm="init" 时检查
- 快速检查文件名是否为 "init.rc"
- 成功注入后立即 unregister kprobe

### 方案二：使用 Manual Hook（推荐）

直接在内核源码中添加 hook 点，避免 kprobe 开销。

需要修改 `fs/read_write.c` 添加 CONFIG_KSU 条件编译。

---

## 推荐方案：Manual Hook

应用 `_kernel_patches/manual_hook/kernel-4.14.patch` 到内核源码。
