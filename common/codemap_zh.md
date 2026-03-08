# OpenHarmony 构建系统 - Common 目录代码映射

## 概述

`common/` 目录包含在所有 OpenHarmony 镜像（system、vendor、ramdisk、updater）中安装的共享运行时库、Sanitizer 配置和必要的系统组件。这些是运行时环境所需的预构建二进制文件和配置文件。

## 目录结构

```
common/
├── BUILD.gn                  # 主构建文件 - 定义 common_packages 组
├── libcpp/                   # C++ 标准库 (libc++)
│   └── BUILD.gn             # libc++_shared.so 安装规则
├── musl/                     # musl libc 运行时
│   ├── BUILD.gn             # musl 链接器和 libc 安装
│   ├── ld-musl-arm.path     # ARM 库搜索路径
│   ├── ld-musl-aarch64.path # ARM64 库搜索路径
│   └── ld-musl-riscv64.path # RISC-V 库搜索路径
├── asan/                     # AddressSanitizer (ASan) 运行时
│   ├── BUILD.gn             # ASan/HWASan/TSan 库安装
│   ├── asan.options         # ASan 运行时选项
│   ├── asan.cfg             # ASan 的 Init 系统配置
│   ├── tsan.options         # ThreadSanitizer 选项
│   └── build_mixed_asan.sh  # 构建混合 ASan 镜像的脚本
└── ubsan/                    # UndefinedBehaviorSanitizer 运行时
    ├── BUILD.gn             # UBSan 库安装
    └── ubsan.cfg            # UBSan init 配置
```

## 1. 主构建文件 (BUILD.gn)

**位置**: `common/BUILD.gn`

**用途**: 定义聚合所有公共运行时依赖的 `common_packages` 组。

**关键组件**:
- **musl_install**: musl libc 链接器和库
- **libcpp_install**: C++ 标准库 (libc++_shared.so)
- **ASan 库**: libclang_rt.asan.so (所有架构)、libclang_rt.hwasan.so、libclang_rt.tsan.so (仅 arm64)
- **ASan/TSan 配置**: asan.options、asan.cfg、tsan.options (启用 Sanitizer 时)
- **UBSan 库**: libclang_rt.ubsan_minimal.so、libclang_rt.ubsan_standalone.so、ubsan.cfg (未启用 Sanitizer 时)

**条件逻辑**:
- HWASan 和 TSan 库仅在 `target_cpu == "arm64"` 时包含
- Sanitizer 配置根据 `is_asan`、`is_tsan` 标志条件包含
- 未启用 ASan/TSan 时使用 UBSan 作为后备

## 2. C++ 标准库 (libcpp/)

**位置**: `common/libcpp/BUILD.gn`

**用途**: 安装用于 C++ 运行时支持的 libc++ 共享库。

**目标**:
- `libcpp_install`: 安装组目标
- `libc++_shared.so`: 来自 Clang 工具链的预构建共享库
  - 支持 arm、arm64、x86_64 架构
  - 当 `use_hwasan == true` 时支持 arm64 HWASan 变体
  - 安装到 `system` 镜像
  - 去除调试符号，生成 mini-debug 信息

**源路径**:
- ARM: `${clang_stl_path}/arm-linux-ohos/libc++_shared.so`
- ARM64: `${clang_stl_path}/aarch64-linux-ohos/libc++_shared.so`
- ARM64+HWASan: `${clang_stl_path}/aarch64-linux-ohos/hwasan/libc++_shared.so`
- x86_64: `${clang_stl_path}/x86_64-linux-ohos/libc++_shared.so`

## 3. musl libc 运行时 (musl/)

**位置**: `common/musl/`

**用途**: 为 OpenHarmony 提供 musl libc 链接器和 C++ 运行时库。

### 3.1 BUILD.gn 目标

**主组**: `musl_install`
依赖:
- `musl-libcxx.so` / `musl-libcxx.so_arm64e`: 用于 musl 的 C++ 标准库
- `musl_ld_path_etc_cfg`: 库搜索路径配置
- `//third_party/musl:musl_libs`: 核心 musl 库
- 架构特定的链接器库

**链接器库**:
- `ld-musl-riscv64.so.1`: RISC-V 64 位动态链接器
- `ld-musl-arm.so.1`: ARM/ARM64 动态链接器
- `ld-musl-arm.so.1_arm64e`: 用于 Apple Silicon 类扩展的 ARM64E 变体

**配置**:
- `musl_ld_path_etc_cfg`: 将库搜索路径文件安装到 `etc/`

### 3.2 库搜索路径文件

这些文件定义动态链接器库搜索路径:

**ld-musl-arm.path** (ARM 32 位):
```
/system/lib:/vendor/lib:/vendor/lib/chipsetsdk:/system/lib/ndk:...
```

**ld-musl-aarch64.path** (ARM 64 位):
```
/system/lib64:/vendor/lib64:/system/lib:/vendor/lib:...
```

**ld-musl-riscv64.path** (RISC-V 64 位):
```
/system/lib64:/vendor/lib64:/system/lib:/vendor/lib:...
```

路径包括 SDK 特定目录 (chipsetsdk、platformsdk、ndk 等)

## 4. AddressSanitizer 运行时 (asan/)

**位置**: `common/asan/`

**用途**: 提供 AddressSanitizer (ASan)、Hardware-assisted AddressSanitizer (HWASan) 和 ThreadSanitizer (TSan) 运行时库和配置。

### 4.1 BUILD.gn 目标

**共享库**:
- `libclang_rt.asan.so`: AddressSanitizer 运行时 (所有架构)
- `libclang_rt.hwasan.so`: 硬件辅助 ASan (仅 arm64)
- `libclang_rt.tsan.so`: ThreadSanitizer 运行时 (仅 arm64)

**配置文件**:
- `asan.options`: 安装到 `system/etc/asan.options`
- `asan.cfg`: 用于设置 ASan 环境的 Init 系统配置
- `tsan.options`: ThreadSanitizer 运行时选项

**安装镜像**:
- ASan: system、ramdisk、updater
- HWASan: system、updater (当 `use_hwasan` 时包含 ramdisk)
- TSan: system、ramdisk、updater

### 4.2 asan.options

AddressSanitizer 的运行时选项:
```
quarantine_size_mb=256
max_redzone=2048
thread_local_quarantine_size_kb=256
detect_odr_violation=0
allocator_may_return_null=1
abort_on_error=0
halt_on_error=1
print_module_map=1
memory_debug=1
heap_history_size_main_thread=1023000
...
```

### 4.3 asan.cfg

用于 ASan/HWASan/TSan 设置的 Init 系统配置:
```json
{
    "jobs": [{
        "name": "pre-init",
        "cmds": [
            "setrlimit RLIMIT_STACK unlimited unlimited",
            "export ASAN_OPTIONS log_path=/dev/asan/asan.log:include=/system/etc/asan.options",
            "export HWASAN_OPTIONS log_path=/dev/hwasan/hwasan.log:include=/system/etc/asan.options",
            "export TSAN_OPTIONS include=/system/etc/tsan.options"
        ]
    }, {
        "name": "early-fs",
        "cmds": [
            "mkdir /data/log/sanitizer/asan/ ...",
            "mkdir /dev/asan/ ...",
            "mount none /data/log/sanitizer/asan /dev/asan bind"
        ]
    }]
}
```

### 4.4 tsan.options

ThreadSanitizer 运行时选项:
```
allow_addr2line=1
allocator_may_return_null=1
detect_deadlocks=1
second_deadlock_stack=1
history_size=7
print_full_thread_history=1
...
```

### 4.5 build_mixed_asan.sh

**用途**: 用于构建"混合"ASan 镜像的脚本，其中某些服务运行 ASan 插桩。

**特性**:
- 同时构建 ASan 和非 ASan 版本
- 创建带有选择性 ASan 启用的 system/vendor 镜像
- 支持自定义服务配置组
- 可将 ASan 二进制文件放置在 `/data/asan` 或原位

**用法**:
```bash
./build_mixed_asan.sh [选项] --gn-args ...
```

**关键操作**:
1. 构建 ASan 变体 → `out.a/`
2. 构建非 ASan 变体 → `out/`
3. 合并带有选择性 ASan 启用的镜像
4. 修改 init 配置以使用 ASan 运行特定服务

## 5. UndefinedBehaviorSanitizer 运行时 (ubsan/)

**位置**: `common/ubsan/`

**用途**: 提供用于检测未定义行为的 UBSan 运行时库 (在未启用 ASan/TSan 时使用)。

### 5.1 BUILD.gn 目标

**组**: `ubsan`
包含:
- `libclang_rt.ubsan_standalone.so`: 完整 UBSan 运行时
- `libclang_rt.ubsan_minimal.so`: 最小 UBSan 运行时
- `ubsan.cfg`: Init 系统配置

**安装**:
- 镜像: system、updater
- 内部 API 标签: platformsdk、chipsetsdk

### 5.2 ubsan.cfg

UBSan init 配置:
```json
{
    "jobs": [{
        "name": "pre-init",
        "cmds": [
            "export UBSAN_OPTIONS print_stacktrace=1:print_module_map=2:log_exe_name=1"
        ]
    }, {
        "name": "post-fs-data",
        "cmds": [
            "mkdir /data/log/sanitizer/ubsan/ 0777 system system"
        ]
    }]
}
```

## 来自其他构建系统组件的依赖

### 对 common/ 的直接依赖

| 文件 | 依赖 | 用途 |
|------|------------|---------|
| `config/BUILD.gn:273` | `//build/common/ubsan:ubsan` | 非 Sanitizer 构建的 UBSan 运行时 |
| `templates/idl/ohos_idl.gni:48` | `//build/common/musl:musl-libcxx.so` | IDL 编译器的 C++ 运行时 |
| `ohos_var.gni:14` | `import("//build/common.gni")` | 公共构建变量 |
| `lite/ohos_var.gni:13` | `import("//build/common.gni")` | 公共构建变量 (Lite) |
| `config/compiler/BUILD.gn:13` | `import("//build/common.gni")` | 公共编译器设置 |

### 对 common_packages 的依赖

`common_packages` 组是主要接口。它通常被以下依赖:
- 镜像生成目标
- 完整系统构建目标
- SDK 打包目标

### 模板依赖

`//build/templates/` 中的各种模板依赖于 `//build/templates/common/` 中的脚本和工具 (注意: 这是与 `//build/common/` 不同的目录):
- `get_subsystem_name.py` - 被 cxx.gni、rust 模板、app.gni 使用
- `check_target.gni` - 被 cxx.gni、rust 模板使用
- `copy.gni` - 被 bpf、ace、预构建模板使用
- `collect_target.gni` - 被 abc、bpf、rust、cxx 模板使用

## 关键概念

### 镜像安装

所有公共组件指定 `install_images` 来控制它们包含在哪些镜像中:
- **system**: 主系统分区
- **vendor**: Vendor 分区
- **ramdisk**: 初始启动 ramdisk
- **updater**: OTA 更新 ramdisk
- **system_arm64e**: ARM64E 变体系统分区

### Sanitizer 选择逻辑

构建系统根据构建标志选择 Sanitizer 运行时:

1. **ASan 构建** (`is_asan = true`):
   - 包含 ASan 运行时库
   - 包含 asan.cfg init 配置
   - 使用 ASan 插桩的 musl 链接器

2. **TSan 构建** (`is_tsan = true`):
   - 包含 TSan 运行时库
   - 包含 asan.cfg (共享配置)

3. **常规构建** (无 Sanitizer):
   - 包含 UBSan 运行时库
   - 包含 ubsan.cfg init 配置

### 架构支持

| 组件 | arm | arm64 | x86_64 | riscv64 |
|-----------|-----|-------|--------|---------|
| libc++_shared.so | ✓ | ✓ | ✓ | - |
| musl 链接器 | ✓ | ✓ | - | ✓ |
| ASan 运行时 | ✓ | ✓ | ✓ | ✓ |
| HWASan 运行时 | - | ✓ | - | - |
| TSan 运行时 | - | ✓ | - | - |
| UBSan 运行时 | ✓ | ✓ | ✓ | ✓ |

### 内部 API 标签

组件声明其 API 稳定性级别:
- `platformsdk`: 平台 SDK API
- `chipsetsdk`: 芯片组 SDK API
- `chipsetsdk_sp`: 芯片组 SDK 服务提供商 API

## 构建集成

### 典型使用流程

1. **构建初始化**: 导入 `common.gni` 获取公共变量
2. **目标定义**: 单个模块可能依赖特定公共目标
3. **镜像生成**: `common_packages` 组包含在镜像依赖中
4. **安装**: 预构建二进制文件复制到适当的镜像目录

### Sanitizer 构建流程

1. 开发者在 GN 参数中设置 `is_asan=true` 或 `is_tsan=true`
2. `common/BUILD.gn` 选择适当的 Sanitizer 库
3. Sanitizer init 配置 (asan.cfg) 安装到 `etc/init/`
4. 启动时，init 系统运行 cfg 中的命令设置环境
5. 从 `/system/etc/asan.options` 加载运行时选项

## 总结

`common/` 目录是 OpenHarmony 构建系统的关键组件，提供:

1. **运行时库**: C++ 标准库 (libc++)、musl libc 链接器
2. **Sanitizer 支持**: ASan、HWASan、TSan、UBSan 运行时及配置
3. **系统配置**: 库搜索路径、Sanitizer 初始化脚本
4. **多架构支持**: arm、arm64、x86_64、riscv64
5. **镜像集成**: 安装到 system、vendor、ramdisk、updater 镜像

这些组件是基础性的 - 没有它们，OpenHarmony 系统无法启动或运行 C/C++ 应用程序。它们架起了工具链 (Clang) 与运行时环境 (musl libc + OpenHarmony init 系统) 之间的桥梁。

(文件结束 - 共 347 行)
