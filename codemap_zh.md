# 仓库图谱：OpenHarmony 构建系统

## 项目职责

这是 **OpenHarmony 构建系统**——一个基于 GN/Ninja 的综合构建框架，用于将 OpenHarmony 源代码编译为系统镜像、SDK 和应用程序包。它提供了 `hb` CLI 工具用于构建编排，并支持标准（富设备）和轻量（IoT/嵌入式）两种构建目标。

## 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                        用户界面                                   │
│                     hb set / hb build / hb clean                 │
└─────────────────────────────────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      构建编排                                     │
│    hb/ → main.py → 模块 → 服务 (Preloader/Loader/GN/Ninja)       │
└─────────────────────────────────────────────────────────────────┘
                                 │
               ┌─────────────────┼─────────────────┐
               ▼                 ▼                 ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   core/gn/      │  │     ohos/       │  │     lite/       │
│  BUILD.gn       │  │  模板与         │  │  轻量构建       │
│  入口点         │  │  打包           │  │  系统           │
└─────────────────┘  └─────────────────┘  └─────────────────┘
               │                 │                 │
               └─────────────────┼─────────────────┘
                                 ▼
┌─────────────────────────────────────────────────────────────────┐
│                      构建执行                                     │
│           GN → Ninja → 编译器 → 镜像/包                          │
└─────────────────────────────────────────────────────────────────┘
```

## 系统入口点

| 文件 | 用途 |
|------|------|
| `hb/__main__.py` | `hb` 命令的 CLI 入口点 |
| `hb/main.py` | 主构建编排器，工作区验证 |
| `core/gn/BUILD.gn` | 主 GN 构建目标定义 |
| `core/gn/dotfile.gn` | GN 点文件配置 |
| `ohos.gni` | 模块的聚合 GNI 导入 |
| `ohos_var.gni` | 全局构建变量定义 |
| `gn_helpers.py` | Python 脚本的 GN 辅助函数 |

## 目录映射（汇总）

| 目录 | 职责摘要 | 详细映射 |
|------|----------|----------|
| `hb/` | **OpenHarmony 构建 CLI 工具** - 基于 Python 的命令行界面，包含 11 个命令模块（build、set、clean、env、tool 等）、参数解析器和构建服务（Preloader、Loader、GN、Ninja、HPM）。实现了外观模式、工厂模式和策略模式。 | [查看映射](hb/codemap.md) |
| `ohos/` | **核心构建模板** - 用于 OpenHarmony 组件构建、打包和镜像生成的 GN 模板。包含 17+ 个子目录：kits、ndk、sdk、app、sa_profile、notice、images、packages、sbom 等。定义如何将部件聚合成系统镜像。 | [查看映射](ohos/codemap.md) |
| `common/` | **运行时库** - 所有 OpenHarmony 镜像上所需的共享运行时组件（libc++、musl、sanitizers）。包含 ASan、HWASan、TSan、UBSan 配置，支持多架构（arm、arm64、x86_64、riscv64）。 | [查看映射](common/codemap.md) |
| `core/` | **GN 入口点** - 连接 `hb` 工具与 GN/Ninja 的中心配置中心。包含主 BUILD.gn 构建目标、dotfile.gn GN 配置，以及 exec_script() 的安全白名单（200+ 条目）。 | [查看映射](core/codemap.md) |
| `lite/` | **轻量构建系统** - 为运行 LiteOS/UniProton 的 IoT/嵌入式设备提供的简化构建。提供 lite_subsystem、lite_component、lite_library 模板和可配置板级工具链（GCC、Clang、IAR）。 | [查看映射](lite/codemap.md) |
| `tools/` | **构建工具** - 30+ 个 Python 脚本，用于依赖分析、组件管理、产品配置迁移和静态代码检查。包含用于 BUILD.gn 验证的 csct.py 和用于可视化的 module_deps_tree.py。 | [查看映射](tools/codemap.md) |
| `scripts/` | **构建自动化** - 46 个 Python 脚本，用作 GN/Ninja 动作处理器。类别包括：入口点、HAP 构建、SDK 生成、代码生成（idl、cargo2gn、bpf）以及 `util/` 中的实用基础设施。 | [查看映射](scripts/codemap.md) |
| `config/` | **配置文件** - JSON 配置文件，用于构建标准、组件白名单、子系统配置和预构建配置。 | [查看映射](config/codemap.md) |
| `templates/` | **构建模板** - GN 模板定义，用于测试框架、IDL、Rust 和其他专用构建场景。 | [查看映射](templates/codemap.md) |
| `misc/` | **杂项工具** - 额外的构建支持工具和脚本。 | [查看映射](misc/codemap.md) |
| `rust/` | **Rust 构建支持** - Rust 特定的构建配置和 cargo 集成。 | [查看映射](rust/codemap.md) |
| `toolchain/` | **工具链定义** - 各种目标架构的交叉编译工具链配置。 | [查看映射](toolchain/codemap.md) |

## 关键配置文件

| 文件 | 描述 |
|------|------|
| `bundle.json` | 构建系统组件描述符 |
| `subsystem_config.json` | 子系统到路径的映射 |
| `ohos.gni` | 模块的通用 GNI 导入 |
| `ohos_var.gni` | 全局构建变量 |
| `test.gni` | 测试框架模板 |
| `common.gni` | 通用构建定义 |
| `version.gni` | 构建系统版本 |
| `prebuilts_config.json` | 预构建二进制配置 |
| `component_compilation_whitelist.json` | 组件构建白名单 |
| `compile_standard_whitelist.json` | 编译器标准白名单 |

## 构建流程

```
1. 产品选择 (hb set)
   └── 用户从 //vendor 或 //product 中选择产品

2. 预加载阶段 (hb/services/preloader.py)
   ├── 解析 product_config.json
   ├── 生成 parts_info.json
   ├── 生成 subsystem_config.json
   └── 输出到 out/{product}/build_configs/

3. 加载阶段 (hb/services/loader.py)
   ├── 处理特性
   ├── 生成 build_vars.json
   └── 准备工具链配置

4. GN 阶段 (hb/services/gn.py → core/gn/)
   ├── 使用 build_configs/ 中的参数执行 gn gen
   ├── 读取 BUILDCONFIG.gn
   ├── 处理 //build/ohos/ 模板
   └── 生成 ninja 文件到 out/{product}/

5. Ninja 阶段 (hb/services/ninja.py)
   └── ninja -C out/{product}/ <targets>

6. 构建后阶段 (hb/services/post_build.py)
   ├── 包生成
   ├── 镜像创建 (ext4, f2fs, cpio)
   ├── SDK 打包（可选）
   └── SBOM 生成（可选）
```

## 集成点

### 外部工具
- **GN** - 生成 Ninja 文件的元构建系统
- **Ninja** - 底层构建执行
- **HPM** - HarmonyOS 包管理器，用于独立构建
- **HDC** - HarmonyOS 设备连接器，用于部署
- **CCache** - 编译器缓存，加快重建速度

### 构建阶段 (hb/services/)
构建分为 12 个阶段：
1. `PRE_BUILD` - 环境设置
2. `PRE_LOAD` - 预加载器执行
3. `LOAD` - 加载器执行
4. `PRE_TARGET_GENERATION` - 目标准备
5. `TARGET_GENERATION` - 目标生成
6. `GN` - GN 元构建
7. `NINJA` - Ninja 构建执行
8. `POST_BUILD` - 后处理
9. `IMAGE_GENERATION` - 系统镜像创建
10. `PACKAGE_GENERATION` - 包创建
11. `SDK_GENERATION` - SDK 打包
12. `INDEP_COMPILATION` - 独立组件构建

### 错误代码分类
错误代码为 4 位数字，其中：
- 第一位表示阶段（1=预加载器，2=加载器，3=GN，4=ninja）
- 剩余位表示具体错误类型

## 使用的设计模式

1. **外观模式** - `hb/__main__.py` 隐藏工作区验证
2. **工厂方法模式** - `hb/main.py` 创建命令模块
3. **依赖注入** - 通过构造函数注入服务
4. **策略模式** - 通过字典映射选择模块
5. **模板方法模式** - `BuildModuleInterface` 定义阶段
6. **单例模式** - `Config` 类用于全局状态
7. **观察者模式** - 构建阶段回调

## 安全特性

- `core/gn/ohos_exec_script_allowlist.gni` - 允许使用 `exec_script()` 的 200+ 路径白名单
- 支持 Sanitizer（ASan、HWASan、TSan、UBSan）用于调试
- 栈保护器和 PIE/PIC 编译标志

## 统计信息

- **跟踪的总文件数**：501
- **Python 脚本**：150+
- **GN/GNI 模板**：100+
- **主目录数**：12
- **构建目标数**：数千个（从产品配置动态生成）

## 快速参考

| 任务 | 命令 | 入口点 |
|------|------|--------|
| 设置产品 | `hb set` | `hb/modules/set.py` |
| 构建 | `hb build` | `hb/modules/build.py` |
| 清理 | `hb clean` | `hb/modules/clean.py` |
| 检查环境 | `hb env` | `hb/modules/env.py` |
| 独立构建 | `hb build --indep` | `hb/modules/indep_build.py` |
