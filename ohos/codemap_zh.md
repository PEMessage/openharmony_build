# OpenHarmony 构建系统 - ohos/ 目录代码地图

## 概述

`ohos/` 目录是 OpenHarmony 特定构建配置、模板和处理流程的核心。它包含 GN（Generate Ninja）构建模板、Python 脚本和配置文件，定义了 OpenHarmony 组件的构建、打包和组装方式。

## 目录结构

```
ohos/
├── ace/              # ArkUI/ACE 框架构建支持
├── app/              # HAP（HarmonyOS Ability Package）构建模板
├── common/           # 通用构建工具和子系统合并
├── ebpf.gni          # eBPF（Extended Berkeley Packet Filter）构建模板
├── hisysevent/       # HiSysEvent 配置处理
├── images/           # 系统镜像生成
├── kernel/           # 内核版本配置
├── kits/             # InnerKits（内部 API）检查
├── native_stub/      # Native stub 库生成
├── ndk/              # NDK（Native Development Kit）构建支持
├── notice/           # 许可证/声明文件收集
├── ohos_kits.gni     # 主 kits 模板定义
├── ohos_part.gni     # OpenHarmony 部件模板
├── ohos_test.gni     # 测试框架模板
├── packages/         # 包组装和安装
├── sa_profile/       # SA（System Ability）配置文件处理
├── sbom/             # SBOM（Software Bill of Materials）生成
├── sdk/              # SDK 打包
├── statistics/       # 构建统计
├── taihe_idl/        # Taihe IDL（Interface Definition Language）支持
├── testfwk/          # 测试框架工具
└── update/           # OTA 更新包生成
```

## 关键组件

### 1. 核心模板（.gni 文件）

#### `ohos_part.gni`
- **用途**：定义 `ohos_part` 模板 - OpenHarmony 部件的基本构建块
- **关键特性**：
  - 将模块列表聚合到部件中
  - 处理 SDK 依赖
  - 生成部件安装信息
  - 支持变体处理（手机等）
  - 与 eBPF 测试集成

#### `ohos_kits.gni`
- **用途**：定义 `ohos_inner_kits` 用于 SDK 库管理
- **关键特性**：
  - 管理 SDK 库（SO、JAR 文件）
  - 处理头文件配置
  - 支持预编译库
  - 接口兼容性检查

#### `ohos_test.gni`
- **用途**：测试聚合模板
- **关键特性**：
  - 按部件分组测试包
  - 管理测试依赖

#### `ebpf.gni`
- **用途**：eBPF 测试用例收集
- **关键特性**：
  - 收集 eBPF 测试用例
  - 生成测试配置

### 2. 子系统目录

#### `kits/`
- **`kits_check.gni`**：用于检查 InnerKits 接口兼容性的模板
- **`kits_check_remove.py`**：验证 SDK 库是否未被移除
- **用途**：通过检查保存的签名确保 API 稳定性

#### `ndk/`
- **`ndk.gni`**：主 NDK 构建模板（615 行）
  - `ohos_ndk_library`：生成 NDK stub 库
  - `ohos_ndk_headers`：处理 NDK 头文件
  - `ohos_ndk_copy`：复制 NDK 文件
  - `ohos_ndk_toolchains`：NDK 工具链管理
  - `ohos_ndk_prebuilt_library`：预编译 NDK 库
  - `current_ndk`：当前 NDK 目标
- **Python 脚本**：
  - `generate_ndk_stub_file.py`：从 JSON 描述生成 stub C 文件
  - `generate_version_script.py`：生成链接器版本脚本
  - `check_ndk_header.py`：验证 NDK 头文件编译
  - `check_ndk_header_signature.py`：检查头文件签名兼容性
  - `collect_ndk_syscap.py`：收集系统能力信息
  - `archive_ndk.py`：将 NDK 归档为 zip 文件
  - `generate_ndk_docs.py`：生成 Doxygen 文档
  - `create_ndk_docs_portal.py`：创建文档门户
  - `copy_notices_file.py`：复制 NDK 声明文件
  - `parse_ndk_targets.py`：从 BUILD.gn 解析 NDK 目标
  - `scan_ndk_targets.py`：扫描仓库中所有 NDK 目标
- **`BUILD.gn`**：主 NDK 构建文件（439 行）
- **`cmake/`**：NDK 的 CMake 工具链文件

#### `sdk/`
- **`sdk.gni`**：SDK 打包模板（405 行）
  - `copy_and_archive`：复制并归档 SDK 模块
  - `make_sdk_modules`：创建 SDK 模块包
  - `make_linux_sdk_modules`：Linux 专用 SDK
  - `make_windows_sdk_modules`：Windows 专用 SDK
  - `make_darwin_sdk_modules`：macOS 专用 SDK
  - `make_ohos_sdk_modules`：OpenHarmony 专用 SDK
  - `current_sdk`：当前 SDK 目标
- **Python 脚本**：
  - `parse_sdk_description.py`：解析 SDK 描述 JSON
  - `generate_all_types_sdk.py`：生成 SDK 构建文件
  - `copy_sdk_modules.py`：复制 SDK 模块
  - `check_sdk_completeness.py`：验证 SDK 完整性
  - `add_notice_file.py`：将声明文件添加到 SDK 归档
  - `convert_permissions.py`：转换权限定义
  - `parse_interface_sdk.py`：解析接口 SDK
  - `parse_public_sdk.py`：解析公共 SDK
  - `generate_hap_build_sdk_config.py`：生成 HAP 构建 SDK 配置
  - `remove_cangjie_ohos_sdk_config.py`：移除仓颉 SDK 配置
  - `parse_description.py`：解析 SDK 描述
- **配置文件**：
  - `ohos_sdk_description_std.json`：标准 SDK 描述
  - `sdk_delivery_list.json`：SDK 交付检查列表
  - `type_to_display_name.json`：SDK 类型显示名称
  - `variant_to_product.json`：变体到产品映射

#### `app/`
- **`app.gni`**：HAP 构建模板（855 行）
  - `ohos_app_scope`：应用范围配置
  - `ohos_assets`：资产管理
  - `ohos_js_assets`：JavaScript 资源
  - `ohos_resources`：资源编译
  - `ohos_app`：主应用目标
  - `ohos_hap`：HAP 包生成
- **`app_internal.gni`**：内部应用模板（751 行）
  - `merge_profile`：合并应用配置文件
  - `compile_resources`：资源编译
  - `package_app`：应用打包
  - `app_sign`：应用签名

#### `sa_profile/`（系统能力配置文件）
- **`sa_profile.gni`**：SA 配置模板（242 行）
  - `ohos_sa_profile`：定义 SA 配置文件
  - `ohos_sa_install_info`：SA 安装信息
  - `ohos_sa_info_archive`：SA 归档处理
- **Python 脚本**：
  - `sa_profile.py`：生成 SA 信息文件
  - `sa_profile_binary.py`：处理二进制 SA 配置文件
  - `sa_profile_merge.py`：合并 SA 配置文件
  - `sa_profile_source.py`：处理源 SA 配置文件
  - `sa_profile_archive.py`：归档 SA 配置文件
  - `src_sa_profile_process.py`：按变体处理 SA 配置文件
- **`sa_info_process/`**：
  - `merge_sa_info.py`：合并 SA 信息 JSON 文件
  - `sort_sa_by_bootphase.py`：按启动阶段对 SA 排序
  - `sa_info_config_errors.py`：错误定义

#### `notice/`
- **`notice.gni`**：声明收集模板（138 行）
  - `collect_notice`：收集模块声明文件
- **Python 脚本**：
  - `collect_module_notice_file.py`：收集单个模块声明
  - `collect_system_notice_files.py`：收集系统范围声明
  - `merge_notice_files.py`：将声明文件合并为最终输出

#### `images/`
- **`BUILD.gn`**：镜像构建配置（430 行）
  - 定义系统、厂商、用户数据、内存磁盘、更新器镜像
- **Python 脚本**：
  - `build_image.py`：主镜像构建脚本（191 行）
  - `get_module_install_dest.py`：计算模块安装目标位置
  - `adlt_wrapper.py`：ADLT（Allowed Dynamic Link Targets）包装器
- **`mkimage/`**：
  - `mkimages.py`：主镜像创建脚本（146 行）
  - `mkextimage.py`：ext4 镜像创建（140 行）
  - `mkf2fsimage.py`：F2FS 镜像创建（155 行）
  - `mkcpioimage.py`：CPIO 内存磁盘创建（140 行）
  - `mkchip_ckm.py`：芯片 CKM 镜像创建
  - `imkcovert.py`：镜像格式转换（稀疏/非稀疏）
  - `judge_updater_image.py`：验证更新器镜像依赖
  - 配置文件：每种镜像类型的 `*_image_conf.txt`

#### `packages/`
- **`BUILD.gn`**：包组装配置（430+ 行）
- **Python 脚本**：
  - `modules_install.py`：将模块安装到系统（347 行）
  - `parts_install_info.py`：生成部件安装信息
  - `resources_collect.py`：收集测试资源
  - `system_notice_info.py`：收集系统声明信息
  - `fs_process.py`：文件系统处理
  - `gen_required_modules_list.py`：生成所需模块列表
  - `generate_host_symlink.py`：生成主机符号链接
  - `system_gzip_package.py`：创建 gzip 包
  - `system_z_package.py`：创建 Z 压缩包
  - `process_field_validate.py`：验证处理字段
  - `check_seccomp_library_name.py`：检查 seccomp 库名称
  - `backup_restore_artifact.py`：备份/恢复工件
  - `bootpath_collection.py`：收集启动路径
  - `kernel_permission.py`：内核权限处理
  - `platforms_install_info.py`：平台安装信息
- **`rules/`**：
  - `categorized_libraries_utils.py`：库分类
  - `categorized-libraries.json`：库类别

#### `testfwk/`
- **`gen_module_list_files.py`**：生成测试模块列表文件
- **`testcase_resource_copy.py`**：复制测试用例资源（349 行）
- **`test_js_file_copy.py`**：复制 JavaScript 测试文件
- **`test_js_stage_file_copy.py`**：复制 JavaScript stage 测试文件
- **`test_py_file_copy.py`**：复制 Python 测试文件
- **`fuzz_config_file_copy.py`**：复制 fuzz 测试配置文件
- **`arkts_tdd_cases_build.py`**：构建 ArkTS TDD 测试用例

#### `hisysevent/`
- **`hisysevent.gni`**：HiSysEvent 模板（64 行）
  - `ohos_hisysevent_install_info`：处理 HiSysEvent 配置
- **`hisysevent_process.py`**：处理 HiSysEvent 配置
- **`gen_def_from_all_yaml.py`**：从 YAML 生成定义（780 行）

#### `common/`
- **`BUILD.gn`**：通用构建目标（126 行）
  - 生成源/二进制安装信息
  - 合并子系统信息
- **`merge_all_subsystem.py`**：合并所有子系统信息
- **`binary_install_info.py`**：二进制安装信息

#### `kernel/`
- **`kernel.gni`**：内核版本配置
  - 定义 `linux_kernel_version`（默认值："linux-6.6"）

#### `native_stub/`
- **`native_stub.gni`**：Native stub 库模板（339 行）
  - `ohos_native_stub_library`：生成 stub 库
  - `ohos_native_stub_versionscript`：生成版本脚本
  - `ohos_native_stub_headers`：处理 stub 头文件

#### `taihe_idl/`
- **`taihe.gni`**：Taihe IDL 模板（103 行）
  - `ohos_taihe`：编译 Taihe IDL 文件
  - `copy_taihe_idl`：复制 Taihe IDL 文件
  - `taihe_shared_library`：构建 Taihe 共享库
- **`taihe_retry.sh`**：Taihe 编译的重试脚本

#### `update/`
- **`check_abi_and_copy_deps.py`**：检查 ABI 兼容性并复制依赖（240 行）

#### `statistics/`
- **`build_overlap_statistics.py`**：计算构建重叠统计（160 行）

#### `ace/`
- **`ace.gni`**：ACE 框架模板（68 行）
  - `js_declaration`：JS 声明处理
  - `gen_js_obj`：生成 JS 对象文件
- **`ace_args.gni`**：ACE 参数配置

#### `sbom/`（软件物料清单）
全面的 SBOM 生成系统：
- **`generate_sbom.py`**：主 SBOM 生成入口（154 行）
- **`README_zh.md`**：详细中文文档（463 行）

**目录结构：**
- **`analysis/`**：依赖分析
  - `depend_graph.py`：依赖图分析器（254 行）
  - `file_dependency.py`：文件依赖分析器（589 行）
  - `install_module.py`：安装模块分析器（138 行）
  - `project_dependency.py`：项目依赖分析器（171 行）

- **`common/`**：通用工具
  - `utils.py`：工具函数（196 行）

- **`converters/`**：SBOM 格式转换器
  - `api.py`：转换器 API（65 行）
  - `base.py`：基础转换器类（104 行）
  - `spdx23.py`：SPDX 2.3 转换器（252 行）

- **`data/`**：数据模型
  - `build_setting.py`：构建设置（33 行）
  - `file_dependence.py`：文件依赖（327 行）
  - `manifest.py`：清单解析器（249 行）
  - `ninja_json.py`：Ninja JSON 模型（59 行）
  - `opensource.py`：开源元数据（81 行）
  - `project_dependence.py`：项目依赖（69 行）
  - `target.py`：构建目标模型（55 行）

- **`extraction/`**：资源提取
  - `local_resource_loader.py`：资源加载器（318 行）
  - `copyright_and_license_scanner.py`：许可证扫描器（349 行）

- **`pipeline/`**：SBOM 生成流水线
  - `sbom_generator.py`：主 SBOM 生成器（377 行）

- **`sbom/`**：SBOM 构建器
  - **`builder/`**：构建器类
    - `base_builder.py`：基础构建器（203 行）
    - `document_builder.py`：文档构建器（341 行）
    - `file_builder.py`：文件构建器（225 行）
    - `package_builder.py`：包构建器（281 行）
    - `relationship_builder.py`：关系构建器（130 行）
    - `sbom_meta_data_builder.py`：元数据构建器（283 行）
  - **`config/`**：配置
    - `field_config.py`：字段配置管理器（169 行）
    - **`configs/`**：JSON 配置
      - `document.config.json`：文档字段
      - `file.config.json`：文件字段
      - `package.config.json`：包字段
      - `relationship.config.json`：关系字段
  - **`metadata/`**：元数据模型
    - `sbom_meta_data.py`：核心 SBOM 元数据（430 行）
  - **`validation/`**：验证
    - `validator.py`：字段验证器（71 行）

## 构建流程

### 1. 部件定义阶段
```
ohos_part("part_name") {
  subsystem_name = "..."
  module_list = ["..."]
}
```
- 部件聚合模块
- 生成部件信息 JSON
- 处理 SDK 依赖

### 2. 模块构建阶段
- 构建单个目标
- 为每个目标生成 module_info.json
- 收集安装元数据

### 3. 安装阶段
```
packages/modules_install.py
```
- 读取所有 module_info.json 文件
- 确定安装路径
- 创建安装清单

### 4. 镜像生成阶段
```
images/build_image.py
```
- 收集已安装文件
- 创建文件系统镜像
- 支持 ext4、f2fs、cpio 格式

### 5. SDK 打包阶段
```
sdk/copy_sdk_modules.py
sdk/parse_sdk_description.py
```
- 解析 SDK 描述
- 将模块复制到 SDK 结构
- 创建平台特定的归档

### 6. SBOM 生成阶段（可选）
```
sbom/generate_sbom.py
```
- 分析构建依赖
- 生成 SPDX 2.3 格式
- 跟踪文件和包关系

## 与更广泛的构建系统集成

### 输入依赖
- `//build/ohos_var.gni`：全局 OpenHarmony 变量
- `//build/config/python.gni`：Python 动作模板
- `${root_build_dir}/build_configs/`：生成的配置文件

### 输出工件
- `${root_build_dir}/packages/`：包输出
- `${root_build_dir}/NOTICE_FILES/`：许可证文件
- `${root_build_dir}/sbom/`：SBOM 文件（启用时）
- `out/{product}/images/`：系统镜像

### 关键集成点
1. **GN 模板**：定义可重用的构建模式
2. **Python 脚本**：处理复杂的构建逻辑
3. **元数据文件**：基于 JSON 的信息交换
4. **配置**：用于构建变体的 GNI 文件

## 使用模式

### 定义部件
```gn
import("//build/ohos/ohos_part.gni")

ohos_part("my_part") {
  subsystem_name = "my_subsystem"
  module_list = [
    "//path/to/module1:target1",
    "//path/to/module2:target2",
  ]
}
```

### 定义 InnerKits
```gn
import("//build/ohos/ohos_kits.gni")

ohos_inner_kits("my_sdk") {
  sdk_libs = [
    {
      type = "so"
      name = "//path/to/lib:libname"
      header = {
        header_files = ["header.h"]
        header_base = "path/to/include"
      }
    },
  ]
}
```

### 构建应用
```gn
import("//build/ohos/app/app.gni")

ohos_hap("my_app") {
  hap_profile = "./module.json"
  sources = ["..."]
}
```

### 生成 SBOM
```bash
./build.sh --product-name {product} --sbom=true
```

## 关键设计原则

1. **模块化**：每个子目录处理特定的关注点
2. **可组合性**：模板可以组合和扩展
3. **配置驱动**：JSON/GNI 文件控制行为
4. **元数据丰富**：构建工件携带安装元数据
5. **多平台**：支持 Linux、Windows、macOS SDK 构建
6. **合规性**：SBOM 生成用于安全和许可证合规
