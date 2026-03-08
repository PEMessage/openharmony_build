# lite/ - OpenHarmony 轻量级构建系统

## 概述

`lite/` 目录包含 OpenHarmony 的**轻量级构建系统**，作为 `hb`（Harmony Build）工具和预加载器的入口点。它提供了一个精简的构建框架，专为运行 LiteOS 或 Linux 内核的资源受限设备（物联网、嵌入式系统）而设计。

## 职责

轻量级构建系统负责以下功能：

1. **构建编排**：管理从产品配置到镜像生成的完整构建流程
2. **交叉编译支持**：为 GCC、Clang 和 IAR ARM 编译器提供工具链抽象
3. **组件管理**：通过 JSON/GN 配置组织子系统和组件
4. **根文件系统创建**：为目标设备生成根文件系统镜像（JFFS2、YAFFS2、VFAT、EXT4）
5. **NDK 生成**：构建用于第三方应用开发的原生开发工具包
6. **HAP 打包**：构建并签名 Harmony Ability 包（HAP）
7. **测试框架集成**：生成测试元数据并管理测试资源
8. **许可证合规**：为第三方组件生成 NOTICE 文件

## 轻量级构建与标准构建的区别

| 方面 | 轻量级构建 | 标准构建 |
|------|------------|----------------|
| **目标设备** | 资源受限的物联网/嵌入式设备 | 富设备（手机、平板等） |
| **内核支持** | liteos_a、liteos_m、Linux、UniProton | Linux（标准） |
| **构建工具** | GN + Ninja 配合轻量级专用模板 | GN + Ninja 配合全功能模板 |
| **组件模型** | 简化的 `lite_component` 模板 | 具有复杂功能的完整 `ohos_part` |
| **工具链** | 板级可配置（GCC/Clang/IAR） | 主要为 Clang/LLVM |
| **Sysroot** | 基于 musl 的自定义 sysroot | 标准 musl libc |
| **输出** | 简化的根文件系统镜像 | 具有复杂分区功能的完整系统镜像 |

### 关键差异详情

1. **GN 构建配置**：
   - 在根目录使用 `BUILDCONFIG.gn` 替代标准 `BUILD.gn`
   - 定义 `ohos_lite = true` 以区分标准构建
   - 简化的目标类型定义（executable、static_library、shared_library、source_set）

2. **子系统/组件模板**：
   - `lite_subsystem` - 定义带有组件列表的子系统
   - `lite_component` - 定义带有特性列表的组件
   - `lite_library` - 统一处理静态/共享/可执行文件的库模板

3. **目标列表生成**：
   - `lite_target_list.gni` 从产品配置动态生成构建目标
   - 基于内核类型和仅用户空间构建过滤目标

## 入口点

### 针对 `hb`（Harmony 构建工具）

`hb` 工具通过这些入口点使用轻量级构建系统：

1. **构建入口**：`//build/lite:ohos`（在 `BUILD.gn` 中定义）
   - 构建所有配置的子系统和组件的主目标
   - 依赖 `lite_target_list` 进行目标枚举

2. **产品入口**：`//build/lite:product`
   - 构建产品特定的配置
   - 引用产品配置中的 `${product_path}`

3. **NDK 入口**：`//build/lite:ndk`
   - 当 `ohos_build_ndk = true` 时构建原生开发工具包

### 针对预加载器

预加载器与以下文件交互：

1. **变量定义**：`ohos_var.gni`
   - 定义所有全局构建变量
   - 从 `${product_config_path}/config.json` 读取产品配置

2. **工具链设置**：`toolchain/BUILD.gn`
   - 基于板级设置配置编译器
   - 支持 GCC、Clang 和 IAR ARM 工具链

## 目录结构

```
lite/
├── BUILD.gn                    # 主构建目标（ohos、product、ndk、prebuilts）
├── ohos_var.gni                # 全局构建变量和产品配置解析
├── lite_target_list.gni        # 从产品配置动态生成目标列表
├── utils.py                    # 构建脚本的通用 Python 工具函数
│
├── config/                     # 构建配置模板和设置
│   ├── BUILDCONFIG.gn          # GN 构建配置（工具链、默认值）
│   ├── BUILD.gn                # 编译器配置（安全、优化、架构）
│   ├── component/              # 组件模板
│   │   └── lite_component.gni  # lite_library、lite_component、build_ext_component 模板
│   ├── subsystem/              # 子系统模板
│   │   ├── lite_subsystem.gni  # lite_subsystem 模板
│   │   ├── aafwk/              #  Ability 框架配置
│   │   ├── graphic/            # 图形子系统配置
│   │   └── hiviewdfx/          # HiView DFX 配置
│   ├── toolchain/              # 工具链配置
│   ├── kernel/                 # 内核特定配置
│   └── test.gni                # 测试配置
│
├── toolchain/                  # 工具链定义
│   ├── BUILD.gn                # 工具链实例化（gcc/clang/iccarm）
│   ├── clang.gni               # Clang 工具链模板
│   ├── gcc.gni                 # GCC 工具链模板
│   └── iccarm.gni              # IAR ARM 工具链模板
│
├── components/                 # 组件元数据
│   └── communication.json      # 通信子系统组件定义
│
├── testfwk/                    # 测试框架支持
│   ├── gen_testfwk_info.py     # 生成测试框架元数据
│   ├── gen_module_list_files.py # 生成测试模块列表
│   └── lite_testcase_resource_copy.py # 测试资源管理
│
├── ndk/                        # 原生开发工具包
│   ├── ndk.gni                 # NDK 模板（ndk_lib、copy_files、ndk_toolchains）
│   ├── BUILD.gn                # NDK 构建目标
│   ├── archive_ndk.py          # NDK 打包脚本
│   ├── build/                  # 独立 NDK 构建系统
│   │   ├── build.py            # NDK 构建入口脚本
│   │   ├── BUILD.gn            # NDK 构建配置
│   │   └── toolchain/          # NDK 工具链配置
│   └── doc/                    # NDK 文档生成
│
├── make_rootfs/                # 根文件系统镜像创建
│   ├── rootfsimg_linux.sh      # Linux 根文件系统镜像构建器
│   ├── rootfsimg_liteos.sh     # LiteOS 根文件系统镜像构建器
│   └── dmverity_linux.sh       # Linux 的 DM-Verity
│
└── [构建脚本]
    ├── build_ext_components.py # 外部组件构建包装器
    ├── hap_pack.py             # HAP 打包和签名
    ├── copy_files.py           # 文件/目录复制工具
    ├── run_shell_cmd.py        # Shell 命令包装器
    └── gen_module_notice_file.py # 第三方许可证文件生成器
```

## 关键模块和职责

### 1. 构建配置（`config/`）

#### `BUILDCONFIG.gn`
- **用途**：核心 GN 构建配置
- **关键功能**：
  - 基于板级配置设置目标操作系统和 CPU
  - 配置工具链（GCC/Clang/IAR ARM）
  - 设置默认编译器标志和配置
  - 定义可执行文件、库和源文件集的默认目标
  - 处理 ccache/xcache 集成

#### `BUILD.gn`（config/）
- **用途**：编译器和链接器配置定义
- **关键配置**：
  - `cpu_arch`：架构特定标志
  - `kernel_macros`：内核类型定义（__LITEOS__、__LINUX__ 等）
  - `security`：安全加固标志（堆栈保护、RELRO、NX 堆栈）
  - `common`：通用编译器标志（-Wall、-fno-common 等）
  - `ohos_clang`：使用 LLD 链接器的 Clang 特定设置
  - `board_config`：板级特定标志和包含路径

#### `lite_component.gni`
- **用途**：轻量级构建系统的核心模板
- **模板**：
  - `lite_library`：统一处理静态、共享和可执行目标的库模板
  - `lite_component`：带有特性列表的组件定义
  - `build_ext_component`：外部构建系统的包装器
  - `ohos_tools`：带有主机特定配置的工具目标
  - `generate_notice_file`：第三方代码的许可证文件生成

#### `lite_subsystem.gni`
- **用途**：子系统组织模板
- **模板**：
  - `lite_subsystem`：将组件分组为子系统
  - `lite_subsystem_test`：子系统的测试变体
  - `lite_subsystem_sdk`：子系统的 SDK 生成
  - `lite_vendor_sdk`：供应商特定的 SDK 生成

### 2. 工具链管理（`toolchain/`）

#### `clang.gni` / `gcc.gni` / `iccarm.gni`
- **用途**：工具链定义模板
- **定义的关键工具**：
  - `cc`：C 编译器
  - `cxx`：C++ 编译器
  - `asm`：汇编器
  - `alink`：静态库归档器
  - `solink`：共享库链接器
  - `link`：可执行文件链接器
  - `stamp`：时间戳文件创建
  - `copy`：文件复制

#### `BUILD.gn`（toolchain/）
- 基于 `board_toolchain_type` 实例化适当的工具链
- 从板级配置设置编译器命令

### 3. 目标列表生成（`lite_target_list.gni`）

- **用途**：从产品配置动态生成构建目标列表
- **流程**：
  1. 读取 `${product_config_path}/config.json`
  2. 读取 `parts_modules_info.json` 获取模块映射
  3. 对于产品配置中的每个子系统：
     - 读取 mini_adapter JSON 获取子系统部件
     - 验证组件存在并支持当前内核
     - 将组件模块列表添加到 `lite_target_list`
  4. 添加设备/产品目标（liteos_m 内核除外）

### 4. 变量定义（`ohos_var.gni`）

- **用途**：全局构建变量
- **关键变量**：
  - `ohos_version`：OpenHarmony 版本字符串
  - `product`、`device_path`、`product_path`：产品/设备路径
  - `ohos_build_type`：debug/release
  - `ohos_kernel_type`：liteos_a、liteos_m、linux、uniproton
  - `ohos_build_*_command`：当前工具链命令
  - `ohos_current_sysroot`：交叉编译的 sysroot 路径
  - `ohos_lite`：设置为 `true` 以标识轻量级构建

### 5. NDK 系统（`ndk/`）

#### `ndk.gni`
- **模板**：
  - `ndk_lib`：将库和头文件复制到 NDK 输出
  - `copy_files`：通用文件/目录复制
  - `ndk_toolchains`：复制工具链二进制文件

#### `BUILD.gn`（ndk/）
- 复制编译器（基于配置的 GCC 或 Clang）
- 复制构建脚本、示例和 sysroot
- 收集来自各子系统的所有 NDK 库
- 创建最终的 NDK zip 压缩包

#### `build/build.py`
- 面向 NDK 用户的独立构建脚本
- 提供 `build` 和 `clean` 命令
- 使用 NDK 中预置的 GN 和 Ninja

### 6. 根文件系统（`make_rootfs/`）

#### `rootfsimg_linux.sh` / `rootfsimg_liteos.sh`
- 为目标设备创建根文件系统镜像
- 支持的文件系统：
  - **JFFS2**：日志闪存文件系统 v2
  - **YAFFS2**：另一种闪存文件系统 v2（仅 LiteOS）
  - **VFAT**：用于 SD 卡的 FAT32
  - **EXT4**：扩展文件系统 v4（仅 Linux）
- 处理设备文件权限和所有权

### 7. HAP 打包（`hap_pack.py` / `hap_pack.gni`）

#### `hap_pack.gni`
- 模板：`hap_pack` - 打包并签名 HAP 文件
- 支持本地签名和基于服务器的签名
- 配置签名算法和证书

#### `hap_pack.py`
- 用于 HAP 打包工作流的 Python 脚本：
  1. 使用 `app_packing_tool.jar` 打包资源
  2. 使用 `hap-sign-tool.jar` 签名包
- 通过环境变量支持远程签名

### 8. 测试框架（`testfwk/`）

#### `gen_testfwk_info.py`
- 生成测试框架元数据 JSON
- 将子系统映射到组件以进行测试执行

#### `gen_module_list_files.py`
- 创建用于测试发现的模块列表文件
- 为测试用例资源生成 `.sources` 文件

#### `lite_testcase_resource_copy.py`
- 基于 XML 配置复制测试资源
- 支持资源目录和构建输出路径

### 9. 实用工具脚本

#### `utils.py`
- Python 构建脚本的通用工具：
  - `exec_command()`：执行带日志记录的 shell 命令
  - `check_output()`：捕获命令输出
  - `read_json_file()`：JSON 文件读取器
  - `makedirs()`：目录创建
  - `CallbackDict`：事件回调系统

#### `build_ext_components.py`
- 构建外部组件的包装器
- 处理预构建步骤和命令执行
- 捕获计时和错误日志

#### `copy_files.py`
- 文件和目录复制工具
- 处理符号链接以及 git/repo 排除

#### `gen_module_notice_file.py`
- 为第三方组件生成许可证 NOTICE 文件
- 读取 `README.OpenSource` 和 `COPYRIGHT.OpenSource` 文件
- 在输出中创建格式化的许可证文件

## 与更广泛构建系统的集成

### 1. 产品配置集成

```
产品配置（config.json）
    ↓
ohos_var.gni（解析 JSON）
    ↓
lite_target_list.gni（生成目标列表）
    ↓
BUILD.gn:ohos 目标（构建所有目标）
```

### 2. 工具链集成

```
板级配置（config.gni）
    ↓
BUILDCONFIG.gn（读取 board_toolchain_*）
    ↓
toolchain/BUILD.gn（实例化工具链）
    ↓
在 ohos_current_*_command 中设置编译器命令
```

### 3. 子系统/组件集成

```
子系统定义（lite_subsystem.gni）
    ↓
组件定义（lite_component.gni）
    ↓
库目标（lite_library 模板）
    ↓
实际构建目标（executable、static_library、shared_library）
```

### 4. 外部构建系统集成

```
外部组件 BUILD.gn
    ↓
build_ext_component 模板
    ↓
build_ext_components.py
    ↓
外部构建命令（make、cmake 等）
```

### 5. NDK 集成

```
原生 API 库（各子系统）
    ↓
ndk_lib 模板
    ↓
ndk/BUILD.gn:ndk_build
    ↓
ndk/BUILD.gn:ndk（归档操作）
    ↓
archive_ndk.py
    ↓
NDK zip 压缩包
```

## 构建流程摘要

1. **配置阶段**：
   - `hb set` 选择产品 → 写入产品配置路径
   - GN 读取 `ohos_var.gni` → 解析产品 `config.json`
   - `lite_target_list.gni` 从产品子系统生成目标列表

2. **生成阶段**：
   - GN 从所有 BUILD.gn 文件生成 build.ninja
   - 基于板级设置配置工具链
   - 将默认配置应用于所有目标

3. **构建阶段**：
   - Ninja 执行 build.ninja
   - `//build/lite:ohos` 构建所有轻量级目标
   - `//build/lite:product` 构建产品特定代码
   - 通过 `build_ext_components.py` 构建外部组件

4. **打包阶段**（可选）：
   - `make_rootfs` 脚本创建文件系统镜像
   - `hap_pack.py` 打包并签名 HAP
   - `archive_ndk.py` 创建 NDK 分发包

5. **输出**：
   - 编译后的二进制文件位于 `$root_out_dir/`
   - 库文件位于 `$root_out_dir/libs/`
   - 用于刷机的根文件系统镜像
   - 面向开发者的 NDK 包

## 安全与合规

### 安全特性
- 堆栈保护（`-fstack-protector-all`）
- 位置无关可执行文件（PIE）
- 共享库的位置无关代码（PIC）
- RELRO（重定位只读）
- NX 堆栈（`-z noexecstack`）
- 立即绑定（`-z now`）

### 许可证合规
- `gen_module_notice_file.py` 跟踪第三方许可证
- third_party 组件需要 `README.OpenSource` 文件
- 为每个输出二进制文件生成 NOTICE 文件
