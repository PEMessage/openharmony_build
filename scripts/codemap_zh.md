# OpenHarmony 构建系统 - Scripts 目录代码地图

## 概述

`scripts/` 目录包含基于 Python 的 OpenHarmony 构建系统自动化工具。这些脚本由 GN（Generate Ninja）构建规则调用，执行各种构建任务，包括编译、打包、签名和代码生成。

## 目录结构

```
scripts/
├── __init__.py                      # 包初始化
├── entry.py                         # 主构建入口点
├── build_target_handler.py          # 构建目标处理
├── ninja_rules_parser.py            # Ninja 构建文件解析器
│
## 应用/HAP 构建
├── hapbuilder.py                    # HAP（HarmonyOS Ability Package）构建器
├── compile_app.py                   # 使用 hvigor 进行应用编译
├── compile_resources.py             # 使用 restool 进行资源编译
├── app_sign.py                      # 应用签名
├── build_js_assets.py               # JavaScript 资源构建
├── generate_js_bytecode.py          # JavaScript 字节码生成
├── ohos_abc.py                      # Ark 字节码（ABC）生成
│
## SDK 生成
├── gen_sdk_build_file.py            # SDK 构建文件生成
├── sign_sdk.py                      # macOS SDK 签名
├── sign_ohos_sdk.py                 # OpenHarmony SDK 签名
├── download_sdk.py                  # 从 CI 下载 SDK
├── interface_mgr.py                 # SDK 接口管理
│
## 代码生成
├── idl.py                           # IDL（接口定义语言）编译器
├── cargo2gn.py                      # Rust Cargo 到 GN 转换器
├── bpf.py                           # BPF（Berkeley Packet Filter）编译
│
## 测试与质量
├── tools_checker.py                 # 构建环境检查器
├── ninja2trace.py                   # 构建跟踪生成
├── summary_ccache_hitrate.py        # CCache 统计
├── gen_subsystem_ebpf_testcase_config.py    # eBPF 测试用例配置
├── gen_summary_ebpf_testcase_config.py      # eBPF 汇总配置
├── generate_test_filter_info.py     # 测试过滤器生成
├── get_warnings.py                  # 警告提取
│
## 工具与辅助
├── copy_ex.py                       # 扩展文件复制
├── find.py                          # 文件查找器
├── get_all_files.py                 # 文件枚举
├── check_file_exist.py              # 文件存在性检查
├── dir_exists.py                    # 目录存在性检查
├── is_substring.py                  # 字符串匹配工具
├── run_shell_cmd.py                 # Shell 命令运行器
├── run_objcopy.py                   # objcopy 包装器
├── run_objcopy_pc_mac.py            # macOS objcopy 包装器
│
## 平台检查
├── check_linux_cpu.py               # Linux CPU 检测
├── check_mac_system_and_cpu.py      # macOS 系统/CPU 检查
├── check_hvigor_hap.py              # Hvigor HAP 验证
│
## 发布与合规
├── code_release.py                  # 开源代码发布
├── merge_notice.py                  # NOTICE 文件合并
├── merge_profile.py                 # Profile 合并
├── collect_publicity.py             # 公开信息收集
│
## 专用处理器
├── kernel_permission_handler.py     # 内核权限注入
├── asan_backup.py                   # ASan 备份处理
│
## 工具子目录
└── util/
    ├── __init__.py                  # 包初始化
    ├── build_utils.py               # 核心构建工具
    ├── file_utils.py                # 文件 I/O 工具
    ├── md5_check.py                 # 基于 MD5 的变更检测
    ├── pycache.py                   # Python 缓存工具
    ├── pyd.py                       # Python 缓存守护进程
    ├── detect_cpu_count.py          # CPU 数量检测
    └── zip_and_md5.py               # ZIP 和 MD5 工具
```

## 核心构建自动化脚本

### entry.py
**用途**: OpenHarmony 构建系统的主入口点。

**用法**: 由顶层构建过程调用，编排整个构建流程。

**关键功能**:
- 解析命令行参数，包括产品名称、目标 CPU、构建目标
- 处理 SDK 构建（`--product-name=ohos-sdk`）
- 支持稀疏镜像、详细模式、快速重建
- 与源码根目录中的 `build.py` 集成

**集成**: 由主构建编排调用（例如 `hb build` 或直接调用）。

---

### build_target_handler.py
**用途**: 处理和验证不同平台的构建目标。

**用法**: 将高级构建目标转换为平台特定的目标。

**关键功能**:
- 读取 `parts_variants.json` 获取目标平台变体
- 支持平台特定构建（手机等）
- 为 ninja 生成 phony 目标名称

**集成**: 在构建设置期间调用以解析目标名称。

---

### ninja_rules_parser.py
**用途**: 解析和更新 Ninja 构建规则以支持多平台。

**用法**: 生成平台特定的构建配置。

**关键功能**:
- 解析 `toolchain.ninja` 文件
- 为不同平台生成 phony 目标
- 使用 subninja 包含更新主 `build.ninja`

**集成**: GN 到 Ninja 构建生成管道的一部分。

---

## 应用构建脚本

### hapbuilder.py
**用途**: 构建和签名 HAP（HarmonyOS Ability Package）文件。

**用法**: `python3 hapbuilder.py --hap-path <输出> --hap-profile <配置> ...`

**关键功能**:
- 打包资源、assets 和原生库
- 使用 Java hapsigner 签名 HAP 文件
- 支持传统模型和 Stage 模型（app_profile 模式）
- 与 Ark 编译器集成进行字节码生成

**集成**: 由 GN 规则调用进行 HAP 目标生成。

---

### compile_app.py
**用途**: 使用 hvigor 构建系统编译应用。

**用法**: `python3 compile_app.py --nodejs <路径> --cwd <项目目录> --sdk-home <sdk> ...`

**关键功能**:
- 设置 Node.js 和 OHPM（OpenHarmony 包管理器）环境
- 运行 `ohpm install` 安装依赖
- 调用 hvigorw 进行实际编译
- 生成未签名 HAP 路径信息
- 支持测试 HAP 构建
- 处理系统库依赖

**集成**: 构建第三方 OpenHarmony 应用的主要脚本。

---

### compile_resources.py
**用途**: 使用 restool 编译应用资源。

**用法**: `python3 compile_resources.py --resources-dir <目录> --restool-path <工具> ...`

**关键功能**:
- 编译资源文件（XML、图片等）
- 生成 `ResourceTable.h` 头文件
- 创建打包资源 ZIP
- 从 profile 提取包名

**集成**: 在 HAP 构建过程中调用进行资源编译。

---

### app_sign.py
**用途**: 签名 HAP/HSP 应用。

**用法**: `python3 app_sign.py --hapsigner <jar> --keystoreFile <路径> --inFile <hap> --outFile <签名后> ...`

**关键功能**:
- 签名单个 HAP 文件
- 从 JSON 列表批量签名多个 HAP
- 支持签名算法选择
- 处理兼容版本

**集成**: HAP 构建过程的最后一步。

---

### generate_js_bytecode.py
**用途**: 使用 es2abc 编译器生成 JavaScript 字节码。

**用法**: `python3 generate_js_bytecode.py --src-js <文件> --dst-file <输出> --frontend-tool-path <路径> ...`

**关键功能**:
- 将 JavaScript/TypeScript 编译为 Ark 字节码（ABC）
- 支持调试信息生成
- 处理模块和 CommonJS 格式
- 支持增量构建的 merge-abc
- 热更新补丁生成

**集成**: 在 JS/TS 资源编译期间调用。

---

### ohos_abc.py
**用途**: Ark 字节码生成的包装器。

**用法**: 与 `generate_js_bytecode.py` 类似，带有 GN 集成。

**关键功能**:
- 适用于 GN 的 es2abc 接口
- 支持合并模式
- 处理模块类型规范

**集成**: ABC 生成的 GN action 脚本。

---

## SDK 生成脚本

### gen_sdk_build_file.py
**用途**: 为 SDK 模块生成 BUILD.gn 文件。

**用法**: `python3 gen_sdk_build_file.py --input-file <json> --sdk-out-dir <目录> ...`

**关键功能**:
- 处理 SDK 模块描述
- 生成预编译库模板
- 支持共享库、JAR 和 Maple 格式
- 处理头文件安装
- 生成接口签名文件

**集成**: SDK 生成管道的一部分。

---

### interface_mgr.py
**用途**: 管理 SDK 接口兼容性检查。

**用法**: `python3 interface_mgr.py --generate --sdk-base-dir <目录> --check_file_dir <目录>`

**关键功能**:
- 为头文件生成 SHA256 签名
- 验证 SDK 接口兼容性
- 创建接口验证的检查文件

**集成**: 用于 SDK 生成以确保 API 兼容性。

---

### sign_sdk.py
**用途**: 为 macOS 分发签名 SDK 二进制文件。

**用法**: `python3 sign_sdk.py --sdk-out-dir <目录>`

**关键功能**:
- 使用 Apple 证书对二进制文件进行代码签名
- 处理 macOS 公证
- 签名特定工具二进制文件（lldb、hdc 等）

**集成**: macOS SDK 打包的最后一步。

---

### download_sdk.py
**用途**: 从 CI 服务器下载预构建的 SDK。

**用法**: `python3 download_sdk.py --branch <名称> --product-name <名称> --api-version <版本>`

**关键功能**:
- 从 CI API 获取每日构建信息
- 下载并解压 SDK 归档
- 处理 OHOS SDK 完整包

**集成**: 用于设置期间获取预构建的 SDK。

---

## 代码生成脚本

### idl.py
**用途**: 编译接口定义语言（IDL）文件。

**用法**: `python3 idl.py --idl-path <工具> --output-archive-path <输出> ...`

**关键功能**:
- 从 IDL 生成 C++、TypeScript 或 Rust 存根
- 支持多种输出语言
- 处理依赖跟踪

**集成**: 为 IPC 接口生成时调用。

---

### cargo2gn.py
**用途**: 将 Rust Cargo 项目转换为 GN 构建文件。

**用法**: `python3 cargo2gn.py --run [--cargo-bin <路径>] [--features <列表>]`

**关键功能**:
- 解析 cargo 构建输出
- 为 Rust crates 生成 BUILD.gn 文件
- 处理依赖和特性
- 支持 build.rs 脚本
- 合并测试 crates

**集成**: 集成 Rust 第三方 crates 时使用。

---

### bpf.py
**用途**: 编译 BPF（Berkeley Packet Filter）程序。

**用法**: `python3 bpf.py --clang-path <路径> --input-file <c文件> --output-file <o文件> ...`

**关键功能**:
- 将 C 编译为 BPF 字节码
- 设置 BPF 目标架构
- 处理包含目录和定义

**集成**: 为 eBPF 程序编译时调用。

---

## 工具脚本

### util/build_utils.py
**用途**: 构建脚本的核心工具函数。

**关键功能**:
- `check_output()`: 执行命令并处理错误
- `temp_dir()`: 临时目录的上下文管理器
- `write_json()`, `read_build_vars()`: 文件 I/O 工具
- `parse_gn_list()`: 解析 GN 列表格式
- `call_and_write_depfile_if_stale()`: 增量构建支持
- `zip_dir()`, `extract_all()`, `merge_zips()`: ZIP 工具
- `add_depfile_option()`, `write_depfile()`: 依赖跟踪
- `expand_file_args()`: 文件参数扩展

**集成**: 几乎所有构建脚本都使用此模块。

---

### util/file_utils.py
**用途**: 文件操作工具。

**关键功能**:
- `find_top()`: 定位仓库根目录
- `read_json_file()`, `write_json_file()`: JSON I/O
- `read_file()`, `write_file()`: 带 GN 格式化的文本文件 I/O

---

### util/md5_check.py
**用途**: 基于 MD5 的增量构建检测。

**关键功能**:
- `call_and_record_if_stale()`: 检查目标是否需要重建
- 跟踪文件和字符串输入变更
- 支持 pycache 集成

---

### util/pyd.py
**用途**: 分布式构建的 Python 缓存守护进程。

**关键功能**:
- `start_server()`: 启动 pycache 守护进程
- `stop_server()`: 停止守护进程
- `show_statistics()`: 缓存命中/未命中统计
- `manage_cache_contents()`: 缓存清理（40GB/15天限制）

---

## 质量与分析脚本

### tools_checker.py
**用途**: 验证构建环境。

**用法**: `python3 tools_checker.py`

**关键功能**:
- 检查操作系统版本（Ubuntu 18.04/20.04/22.04）
- 验证必需的包已安装
- 使用 `build_package_list.json` 获取包列表

---

### ninja2trace.py
**用途**: 将 Ninja 日志转换为 Chrome 跟踪格式。

**用法**: `python3 ninja2trace.py --ninja-log <文件> --trace-file <输出> --duration-file <输出>`

**关键功能**:
- 解析 `.ninja_log` 文件
- 生成兼容 Chrome 跟踪查看器的 JSON
- 计算构建持续时间统计

---

### code_release.py
**用途**: 打包开源代码用于发布。

**用法**: `python3 code_release.py --output <tar.gz> --root-dir <目录> --scan-dirs <目录> --scan-licenses <许可证>`

**关键功能**:
- 扫描 `README.OpenSource` 文件
- 按许可证类型过滤
- 创建可发布代码的 tar 包

---

## 与构建系统集成

### GN 集成

大多数脚本设计为 GN 中的 `action()` 目标调用：

```gn
action("generate_hap") {
  script = "//build/scripts/hapbuilder.py"
  inputs = [ ... ]
  outputs = [ ... ]
  args = [ ... ]
}
```

### Depfile 支持

脚本支持 Ninja depfiles 以实现正确的增量构建：
- 使用 `build_utils.add_depfile_option(parser)` 添加 `--depfile` 参数
- 使用 `build_utils.call_and_write_depfile_if_stale()` 自动生成 depfile

### 常见模式

1. **参数解析**: 所有脚本使用 `argparse` 或 `optparse`
2. **错误处理**: 使用 `build_utils.CalledProcessError` 处理命令失败
3. **日志**: 将进度消息打印到 stdout
4. **退出码**: 成功返回 0，失败返回非零

## 总结

`scripts/` 目录是 OpenHarmony 构建系统的核心自动化层，提供：

- **46 个 Python 脚本**用于各种构建任务
- `util/` 子目录中的 **8 个工具模块**
- 支持多种语言（C/C++、Rust、JavaScript、Java）
- 跨平台支持（Linux、macOS）
- 通过 MD5 检查和 depfiles 支持增量构建
- 与 GN/Ninja 构建系统集成
- SDK 生成和签名功能
- HAP 构建和签名用于应用分发
