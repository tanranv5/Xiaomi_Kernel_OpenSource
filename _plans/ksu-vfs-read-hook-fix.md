# KernelSU/SukiSU 模块安装问题解决方案

## 问题分析

### 核心问题
- **现象**：开启 vfs_read hook 会卡 MI（小米 Logo）；禁用 vfs_read hook 能开机但模块无法安装
- **根因**：ksud 未生效/未注入

---

## 为什么必须搞 vfs_read hook？

### vfs_read hook 的核心作用

在 init 进程读取 `/system/etc/init/hw/init.rc` 时，动态注入以下 RC 脚本：

```
on post-fs-data
    exec u:r:su:s0 root -- /data/adb/ksud post-fs-data

on nonencrypted
    exec u:r:su:s0 root -- /data/adb/ksud services

on property:sys.boot_completed=1
    exec u:r:su:s0 root -- /data/adb/ksud boot-completed
```

### ksud 的完整职责（不仅仅是模块挂载）

| 阶段 | ksud 命令 | 执行的操作 |
|------|-----------|------------|
| **post-fs-data** | `ksud post-fs-data` | 1. 加载 allowlist<br>2. 启动 observer<br>3. 处理模块更新/删除<br>4. 加载 sepolicy.rule<br>5. 执行 post-fs-data.sh<br>6. 加载 system.prop<br>7. **执行 metamodule 挂载脚本** |
| **services** | `ksud services` | 执行 service.sh 脚本 |
| **boot-completed** | `ksud boot-completed` | 执行 boot-completed.sh 脚本 |

### 没有 vfs_read hook 的后果

1. ksud 守护进程不会启动
2. 模块脚本不执行
3. **模块挂载不发生**（metamodule 的 `metamount.sh` 不执行）
4. sepolicy 规则不加载
5. system.prop 不生效
6. **结果**：模块"安装"了但实际不生效

---

## 方案对比

### 方案 A：优化 kprobe vfs_read hook（推荐尝试）

| 项目 | 说明 |
|------|------|
| **触发时机** | 仍使用 kprobe hook `__arm64_sys_read`，但添加快速过滤 |
| **路径过滤** | 在 handler 最开始检查 `current->pid != 1` 直接返回 |
| **优点** | 无需修改内核源码，改动最小 |
| **缺点** | kprobe 本身仍有开销，可能仍有兼容性问题 |

**具体改法**（修改 `ksud.c:511-518`）：
```c
static int sys_read_handler_pre(struct kprobe *p, struct pt_regs *regs)
{
    // 快速路径：仅处理 init 进程
    if (likely(current->pid != 1))
        return 0;
    if (likely(strcmp(current->comm, "init")))
        return 0;
    
    struct pt_regs *real_regs = PT_REAL_REGS(regs);
    unsigned int fd = PT_REGS_PARM1(real_regs);
    ksu_handle_sys_read(fd);
    return 0;
}
```

### 方案 B：Manual Hook（内核源码修改）

| 项目 | 说明 |
|------|------|
| **触发时机** | 编译时静态插入到 `vfs_read()` 函数内部 |
| **路径过滤** | `unlikely(ksu_vfs_read_hook)` + pid=1 + filename="init.rc" |
| **优点** | 性能最优，无运行时开销 |
| **缺点** | 需要 patch 内核源码 |

**需要应用的 patch**：`_kernel_patches/manual_hook/kernel-4.14.patch`

### 方案 C：手动替代 ksud（你的补充问题）

**可行性分析**：

| 功能 | 能否手动替代 | 说明 |
|------|--------------|------|
| 模块脚本执行 | ✅ 可以 | 手动在 init.rc 添加 exec 命令 |
| 模块挂载 | ⚠️ 部分可以 | 需要手写 mount 脚本 |
| sepolicy 加载 | ❌ 困难 | 需要 ksud 的 sepolicy 接口 |
| allowlist 管理 | ❌ 不行 | 需要内核模块支持 |
| su 权限授予 | ❌ 不行 | 需要内核模块的 supercall |

**手动方案的限制**：

1. **SELinux 权限问题**：
   - 手动脚本需要 `u:r:su:s0` 上下文才能执行
   - 这个上下文由 KSU 内核模块在 `init second_stage` 时注入
   - 如果 vfs_read hook 不工作，sepolicy 规则也不会被注入

2. **模块挂载的复杂性**：
   - SukiSU 使用 metamodule 系统（`metamount.sh`）
   - 默认挂载方式是 overlay/bind mount
   - 需要正确的 SELinux 上下文

3. **缺失的功能**：
   - 无法使用 Manager App 管理模块
   - 无法使用 su 命令（需要 allowlist）
   - 无法动态加载 sepolicy 规则

**手动方案的可行路径**：

如果你只需要让**特定脚本**在启动时执行，可以：

```bash
# 1. 修改 init.rc（需要解包 boot.img）
# 在 on post-fs-data 段添加：
on post-fs-data
    exec u:r:init:s0 root -- /data/adb/my_script.sh

# 2. 手动挂载模块（在脚本中）
mount -o bind /data/adb/modules/xxx/system/app /system/app
```

**风险**：
- `u:r:init:s0` 上下文权限有限
- 无法访问 /data 分区（post-fs-data 阶段 /data 可能未解密）
- dm-verity 可能阻止修改 system 分区

---

## 推荐方案

### 首选：方案 A（优化 kprobe）

**理由**：
1. 改动最小，只需修改 `ksud.c` 一个文件
2. 不需要重新 patch 内核源码
3. 可以快速验证是否解决卡 MI 问题

**如果方案 A 仍然卡 MI**：

### 备选：方案 B（Manual Hook）

**理由**：
1. 完全避免 kprobe 开销
2. 与 SukiSU 官方推荐方式一致
3. 长期稳定

---

## 关于手动替代方案的结论

**不推荐**，因为：

1. **SELinux 是最大障碍**：没有 KSU 内核模块注入的 sepolicy 规则，手动脚本几乎无法执行任何有意义的操作

2. **ksud 的职责远不止模块挂载**：
   - allowlist 管理（决定哪些 app 可以获得 root）
   - supercall 接口（su 命令的实现）
   - sepolicy 动态加载
   - 模块配置系统

3. **如果只是想让脚本执行**：
   - 可以考虑使用 Magisk 的 init.d 方式
   - 或者直接修改 boot.img 中的 init.rc

---

## 验证方法

1. **启动测试**：刷入后观察是否卡 MI
2. **dmesg 检查**：
   ```bash
   adb shell dmesg | grep -E "ksu|ksud|init.rc|read_proxy"
   ```
3. **模块安装测试**：
   ```bash
   adb shell /data/adb/ksud module install /sdcard/module.zip
   ```
