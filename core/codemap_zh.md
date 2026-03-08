# core/ - OpenHarmony 构建系统核心

**位置:** `/home/zhuojw/a_git/ohos-mani-v2/build/core`

## 总体职责

`core/` 目录是 OpenHarmony 基于 GN 的构建系统的**中央入口点和配置中心**。它作为 `hb` 构建工具与底层 GN/Ninja 构建引擎之间的桥梁，定义了：

1. **构建配置根目录** - 指向主构建配置的 `.gn` 文件
2. **构建入口点** - 编排整个构建过程的 GN 目标
3. **安全与沙盒** - 用于构建安全的脚本执行白名单
4. **构建工具** - 用于构建验证的支持脚本

core/ 目录在 GN 启动时加载，并为系统中所有其他构建文件配置整个构建上下文。

---

## 目录结构

```
core/
├── gn/
│   ├── BUILD.gn                    # 主构建目标和编排
│   ├── dotfile.gn                  # GN 点文件 - 构建系统入口点
│   └── ohos_exec_script_allowlist.gni  # 安全: 允许的脚本执行路径
└── build_scripts/
    └── verify_notice.sh            # 许可证声明验证脚本
```

---

## 1. GN 点文件配置

### 文件: `gn/dotfile.gn`

这是 GN 启动构建时首先读取的 **GN 点文件**（相当于仓库根目录的 `.gn`）。

**关键配置:**

| 变量 | 值 | 用途 |
|----------|-------|---------|
| `buildconfig` | `//build/config/BUILDCONFIG.gn` | 主构建配置文件 |
| `root` | `//build/core/gn` | 构建系统的源代码根位置 |
| `script_executable` | `/usr/bin/env` | action/exec_script 的默认解释器 |
| `ohos_components_support` | `true` | 启用 OpenHarmony 组件系统 |
| `exec_script_whitelist` | (来自 `ohos_exec_script_config`) | 脚本执行的安全白名单 |

**导入链:**
```
dotfile.gn
    ↓ (导入)
ohos_exec_script_allowlist.gni
    ↓ (定义)
ohos_exec_script_config.exec_script_allowlist
```

---

## 2. 构建入口点 (BUILD.gn)

### 文件: `gn/BUILD.gn`

这是定义顶层构建目标的 **根 BUILD.gn**。它是 GN 初始化后构建过程的主入口点。

### 构建目标概览

#### 目标选择逻辑

构建根据 `product_name` 选择目标：

```
if (product_name == "ohos-sdk"):
    → build_ohos_sdk
else if (product_name == "arkui-x"):
    → arkui_targets
else:
    → make_all (标准系统构建)
```

### 主要构建目标

#### 1. `build_ohos_sdk` (用于 SDK 构建)

**用途:** 构建 OpenHarmony SDK 包

**依赖项:**
- `//build/ohos/ndk:ohos_ndk` - 原生开发工具包
- `//build/ohos/sdk:ohos_sdk` - SDK 生成
- `//build/ohos/sdk:ohos_sdk_verify` - SDK 验证

#### 2. `arkui_targets` (用于 ArkUI-X 构建)

**用途:** 构建跨平台 ArkUI-X SDK

**依赖项:**
- `//build_plugins/sdk:arkui_cross_sdk`

#### 3. `make_all` (标准系统构建)

**用途:** 构建 OpenHarmony 系统镜像的主目标

**依赖项:**
- `:make_inner_kits` - 构建内部组件接口
- `:packages` - 包生成
- `:images` (条件性) - 系统镜像创建（仅限标准系统）

**条件:**
```gn
if (is_standard_system && !is_llvm_build) {
  deps += [ ":images" ]
}
```

### 子目标

| 目标 | 用途 | 条件 |
|--------|---------|-----------|
| `:images` | 创建系统镜像 | `!is_llvm_build` |
| `:packages` | 生成可安装包 | 始终 |
| `:make_inner_kits` | 构建组件内部接口 | 始终 |
| `:build_all_test_pkg` | 构建测试包 | `testonly = true` |
| `:make_test` | 测试用例打包 | `testonly = true` |

### 目标路径

```
:images           → //build/ohos/images:make_images
:packages         → //build/ohos/packages:make_packages
:make_inner_kits  → $root_build_dir/build_configs:inner_kits
:make_test        → //build/ohos/packages:build_all_test_pkg + 测试打包
```

---

## 3. 安全配置

### 文件: `gn/ohos_exec_script_allowlist.gni`

此文件定义了构建过程中**脚本执行的安全白名单**。

**用途:**
- 限制哪些 BUILD.gn 文件可以使用 `exec_script()`
- 防止构建过程中的任意脚本执行
- 增强构建的可重现性和安全性

**结构:**
```gni
ohos_exec_script_config = {
  exec_script_allowlist = [
    "//arkcompiler/ets_frontend/ts2panda/BUILD.gn",
    "//base/hiviewdfx/hiview/BUILD.gn",
    "//build/config/BUILDCONFIG.gn",
    "//foundation/arkui/ace_engine/ace_config.gni",
    // ... 200+ 条目
  ]
}
```

**允许的脚本类别:**

| 类别 | 示例 |
|----------|----------|
| 构建系统 | `//build/config/BUILDCONFIG.gn`, `//build/ohos_var.gni` |
| 编译器/工具链 | `//build/toolchain/BUILD.gn`, `//build/config/compiler/*` |
| 模板 | `//build/templates/cxx/cxx.gni`, `//build/templates/rust/*` |
| ArkCompiler | `//arkcompiler/ets_frontend/*`, `//arkcompiler/runtime_core/*` |
| 基础系统 | `//foundation/arkui/*`, `//foundation/communication/*` |
| 第三方 | `//third_party/flutter/*`, `//third_party/skia/*` |
| 设备/板级 | `//device/board/*`, `//device/soc/*` |

**与 dotfile.gn 的集成:**
```gni
import("//build/core/gn/ohos_exec_script_allowlist.gni")
exec_script_whitelist = ohos_exec_script_config.exec_script_allowlist
```

---

## 4. 构建脚本

### 文件: `build_scripts/verify_notice.sh`

**用途:** 验证 NOTICE 文件格式以确保许可证合规

**用法:**
```bash
verify_notice.sh <notice_file> <output_file> <platform_dir>
```

**参数:**
- `$1`: 要验证的 NOTICE 文件路径
- `$2`: 验证结果输出文件（"Success" 或 "Failed"）
- `$3`: 包含辅助文件的目录

**验证逻辑:**
1. 检查 NOTICE 文件是否存在（如果不存在，返回 Success）
2. 统计特定分隔符行数：
   - `====` 分隔行
   - "Notices for file(s):" 标记
   - `----` 分隔线
3. 验证计数是否匹配（等号线数量必须等于文件标记行数量）
4. 将 "Success" 或 "Failed" 输出到结果文件

**使用者:** 包生成用于验证第三方许可证声明

---

## 5. 构建流程

### 初始化流程

```
1. 调用 hb build 命令
   ↓
2. hb/main.py → _init_build_module()
   ↓
3. OHOSPreloader.run() - 加载产品/设备配置
   ↓
4. OHOSLoader.run() - 生成构建配置
   ↓
5. Gn.run() - 执行 'gn gen'
   ↓
6. GN 读取 //build/core/gn/dotfile.gn
   ↓
7. dotfile.gn 导入 BUILDCONFIG.gn 并设置构建上下文
   ↓
8. GN 解析 //build/core/gn/BUILD.gn (根 BUILD.gn)
   ↓
9. BUILD.gn 导入 //build/ohos_var.gni
   ↓
10. 基于 product_name 解析目标
   ↓
11. Ninja.run() - 执行 'ninja' 构建目标
```

### 目标解析流程

```
hb build
  ↓
Gn.execute_gn_gen_cmd()
  ↓
gn gen --args="..." <out_path>
  ↓
GN 读取 dotfile.gn
  ↓
GN 加载 BUILDCONFIG.gn (buildconfig)
  ↓
GN 解析 //build/core/gn/BUILD.gn
  ↓
基于产品的目标选择:
    - ohos-sdk → :build_ohos_sdk
    - arkui-x  → :arkui_targets
    - default  → :make_all
  ↓
:make_all 依赖项:
    ├─ :make_inner_kits
    ├─ :packages
    └─ :images (如果是标准系统)
```

---

## 6. 与 hb 工具的集成

### hb → core/ 集成点

| hb 组件 | core/ 文件 | 用途 |
|--------------|------------|---------|
| `hb build` | `gn/BUILD.gn` | 执行主构建目标 |
| `hb build` | `gn/dotfile.gn` | GN 读取此文件进行构建配置 |
| GN 服务 | `gn/dotfile.gn` | `Gn._execute_gn_gen_cmd()` 调用 gn |
| Ninja 服务 | 所有构建目标 | `Ninja._execute_ninja_cmd()` 执行构建 |

### GN 命令执行

来自 `hb/services/gn.py`:
```python
def _execute_gn_gen_cmd(self):
    gn_gen_cmd = [
        self.exec, 'gen',
        '--json=gn_log.json',
        '--args={}'.format(' '.join(self._convert_args())),
        self.config.out_path
    ] + self._convert_flags()
```

此命令:
1. 在 `//build/core/gn` 处使用点文件运行 `gn gen`
2. 从 hb 配置传递构建参数
3. 在 `out_path` 中生成 ninja 构建文件

### 构建目标执行

来自 `hb/services/ninja.py`:
```python
def _execute_ninja_cmd(self):
    ninja_cmd = [
        self.exec,
        '-w', 'dupbuild=warn',
        '-C', self.config.out_path
    ] + self._convert_args()
```

Ninja 执行 `//build/core/gn/BUILD.gn` 中定义的目标。

---

## 7. 依赖关系

### core/ → 其他构建模块

```
core/gn/BUILD.gn:
  ├─ 导入 //build/ohos_var.gni (构建变量)
  ├─ 依赖 //build/ohos/ndk:ohos_ndk
  ├─ 依赖 //build/ohos/sdk:ohos_sdk
  ├─ 依赖 //build/ohos/images:make_images
  ├─ 依赖 //build/ohos/packages:make_packages
  └─ 依赖 $root_build_dir/build_configs:inner_kits

core/gn/dotfile.gn:
  ├─ 导入 //build/core/gn/ohos_exec_script_allowlist.gni
  ├─ 引用 //build/config/BUILDCONFIG.gn
  └─ 设置 root = "//build/core/gn"
```

### 构建系统导入链

```
dotfile.gn
    ↓
BUILDCONFIG.gn (主配置)
    ↓
    ├─ //build/common.gni
    ├─ //build/version.gni
    └─ //build/ohos_var.gni (由 BUILD.gn 导入)
        ↓
        ├─ 产品配置 (来自预加载器)
        ├─ 设备配置
        └─ 工具链配置
```

---

## 8. 关键变量和配置

### 来自 BUILDCONFIG.gn (由 core/ 引用)

| 变量 | 描述 | 用于 |
|----------|-------------|---------|
| `product_name` | 目标产品名称 | BUILD.gn 目标选择 |
| `device_name` | 目标设备名称 | 设备特定配置 |
| `is_standard_system` | 标准操作系统构建标志 | 条件目标 |
| `is_llvm_build` | LLVM 工具链构建 | 如果为真则跳过镜像 |
| `is_mini_system` | 迷你操作系统构建标志 | 轻量系统配置 |
| `is_small_system` | 小型操作系统构建标志 | 轻量系统配置 |
| `ohos_indep_compiler_enable` | 独立组件构建 | 功能标志 |

### 来自 ohos_var.gni (由 BUILD.gn 导入)

| 变量 | 描述 | 示例 |
|----------|-------------|---------|
| `system_base_dir` | 系统包输出目录 | "system" |
| `ramdisk_base_dir` | 内存磁盘输出目录 | "ramdisk" |
| `vendor_base_dir` | 厂商包目录 | "vendor" |
| `build_ohos_sdk` | 构建 SDK 标志 | true/false |
| `build_ohos_ndk` | 构建 NDK 标志 | true/false |

---

## 9. 文件权限和安全性

### 脚本执行安全

`ohos_exec_script_allowlist.gni` 实现了**白名单方式**：

1. 只有列出的 BUILD.gn 文件可以使用 `exec_script()`
2. 从未列出文件执行脚本的尝试将失败
3. 这可以防止恶意或意外的脚本执行

### 构建沙盒支持

来自 BUILDCONFIG.gn:
```gni
declare_args() {
  use_sandbox = false
  sandbox_debug = false
}
```

启用后，action 目标将在沙盒环境中运行以获得额外的安全性。

---

## 总结

`core/` 目录是 **OpenHarmony 构建系统的基础**：

1. **入口点**: `dotfile.gn` 是 GN 读取的第一个文件
2. **配置**: 指向主构建配置 (BUILDCONFIG.gn)
3. **安全**: 实现脚本执行白名单
4. **编排**: 在 BUILD.gn 中定义顶层构建目标
5. **集成**: 将 hb 工具与 GN/Ninja 构建引擎桥接

当您运行 `hb build` 时，执行流程通过 core/ 来初始化构建上下文，基于产品配置选择适当的目标，并委托给底层构建系统。
