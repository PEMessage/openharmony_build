# OpenHarmony 构建工具 - 综合代码地图

## 概述

`tools/` 目录包含各种基于 Python 的实用程序和脚本，用于支持 OpenHarmony 构建系统。这些工具处理组件管理、依赖分析、静态代码检查和产品配置转换。

## 目录结构

```
tools/
├── readme.md                              # 基本使用文档（中文）
├── product_config_version_convert.py      # 产品配置格式转换器
├── check_deps/                            # 依赖使用检查器
│   ├── check_deps.py
│   └── README.md
├── module_dependence/                     # 模块依赖分析
│   ├── dependence_analysis.py
│   ├── module_deps.py
│   ├── module_deps_tree.py
│   ├── part_deps.py
│   ├── file_utils.py
│   ├── README.md
│   └── codemap.md
└── component_tools/                       # 组件管理工具
    ├── generate_kconfig.py
    ├── parse_kconf.py
    ├── component_node.py
    ├── components_dependence_analysis.py
    ├── full_components_generator.py
    ├── kconfig                           # Kconfig 示例文件
    ├── codemap.md
    └── static_check/                     # 组件静态检查工具（CSCT）
        ├── csct.py                       # 主入口点
        ├── csct_online.py                # 在线/CI 版本
        ├── csct_online_entry.py          # 在线检查的入口包装器
        ├── csct_online_prehandle.py      # PR 差异预处理器
        ├── readme.md                     # 文档和规则
        ├── config/
        │   └── csct_whitelist.conf       # 白名单目录
        ├── gn_check/                     # GN 文件验证
        │   ├── check_gn.py
        │   ├── check_gn_online.py
        │   ├── gn_common_tools.py
        │   └── readme.md
        └── bundle_check/                 # bundle.json 验证
            ├── bundle_json_check.py
            ├── bundle_check_online.py
            ├── bundle_check_common.py
            ├── get_subsystem_with_component.py
            ├── warning_info.py
            └── readme.md
```

---

## 1. 产品配置转换器

### 文件：`product_config_version_convert.py`

**用途**：将产品配置文件从旧格式（2.0）转换为 OpenHarmony 使用的新 3.0 格式。

**用法**：
```bash
python3 product_config_version_convert.py {product_name}.json
```

**关键函数**：
- `merge_files()`：合并产品和设备配置文件
- `readjson()`：读取配置并转换为 3.0 格式
- `merge()`：合并两个字典

**集成**：用于产品迁移期间，使现有产品适应新的构建系统要求。

---

## 2. 依赖检查器（`check_deps/`）

### 文件：`check_deps.py`

**用途**：验证 BUILD.gn 文件中的模块依赖是否正确使用 `deps`（内部）与 `external_deps`（跨组件）。

**解决的问题**：开发人员有时错误地将跨组件依赖放在 `deps` 中而不是 `external_deps` 中，违反了组件架构。

**用法**：
```bash
# 首先，启用 check_deps 进行构建
./build.sh --product-name {product} --gn-args check_deps=true --build-only-gn

# 然后运行检查器
python3 build/tools/check_deps/check_deps.py \
    --parts-path-file out/{product}/build_configs/parts_info/parts_path_info.json \
    --deps-path out/{product}/deps_files
```

**输出**：`wrong_used_deps.json` - 列出依赖分类错误的模块。

**集成**：与 GN 构建输出配合使用，分析依赖声明。

---

## 3. 模块依赖分析（`module_dependence/`）

此包提供模块和组件级别的全面依赖分析。

### 3.1 核心分析模块

**文件：`dependence_analysis.py`**
- 读取 GN 生成的模块依赖文件
- 解析外部依赖
- 合并内部和外部依赖标签
- **关键函数**：`get_all_deps_data()` - 返回完整的依赖图

### 3.2 模块依赖

**文件：`module_deps.py`**

**用途**：生成模块级别的依赖信息。

**用法**：
```bash
python3 build/tools/module_dependence/module_deps.py \
    --deps-files-path out/{product}/deps_files
```

**输出**：
- `all_deps_data.json` - 带有标签的原始依赖数据
- `module_deps_info.json` - 整合的模块依赖

### 3.3 组件依赖

**文件：`part_deps.py`**

**用途**：生成组件级别的依赖信息和可视化。

**用法**：
```bash
python3 build/tools/module_dependence/part_deps.py \
    --deps-files-path out/{product}/deps_files \
    --graph  # 可选：生成 HTML 依赖图
```

**输出**：
- `part_deps_info.json` - 组件级别的依赖
- `part-deps-graph.html` - 交互式依赖可视化（需要 pyecharts）

### 3.4 模块依赖树

**文件：`module_deps_tree.py`**

**用途**：为特定模块的依赖生成树形可视化。

**用法**：
```bash
python3 build/tools/module_dependence/module_deps_tree.py \
    --module-name {part_name}:{module_name} \
    --module-deps-file out/{product}/module_deps_info/module_deps_info.json
```

**输出**：`{part_name}__{module_name}.html` - 交互式树形图

### 3.5 实用工具

**文件：`file_utils.py`**
- JSON 文件读写工具
- 处理编码和错误情况

**集成**：所有工具都需要使用 `check_deps=true` 标志的 GN 构建输出。

---

## 4. 组件工具（`component_tools/`）

### 4.1 Kconfig 生成器

**文件：`generate_kconfig.py`**

**用途**：从产品配置生成 Kconfig 文件，用于菜单驱动的组件选择。

**用法**：
```bash
python3 generate_kconfig.py \
    --base_product=productdefine/common/base/base_product.json \
    --outdir=./
```

**输出**：用于 kconfig-frontends 的 `kconfig` 文件

### 4.2 Kconfig 解析器

**文件：`parse_kconf.py`**

**用途**：将 `.config` 文件（从 Kconfig 生成）解析回 JSON 产品配置。

**用法**：
```bash
python3 parse_kconf.py \
    --deps=out/{product}/part_deps_info/part_deps_info.json \
    --base_product=productdefine/common/base/base_product.json \
    --config=./.config \
    --out=./product.json
```

**关键特性**：自动将传递依赖添加到配置中。

### 4.3 组件节点模型

**文件：`component_node.py`**

**用途**：组件和模块表示的数据模型。

**类**：
- `Module`：表示具有 deps/external_deps 的单个构建目标
- `Node`：表示包含多个模块的组件（part）

**用法**：由 `components_dependence_analysis.py` 用于解析 BUILD.gn 文件。

### 4.4 组件依赖分析

**文件：`components_dependence_analysis.py`**

**用途**：分析 BUILD.gn 文件并使用 Graphviz 生成组件依赖图。

**用法**：
```bash
python3 components_dependence_analysis.py \
    --root-path {ohos_root} \
    --output {output_path}
```

### 4.5 完整组件生成器

**文件：`full_components_generator.py`**

**用途**：生成包含系统中所有可用组件的 `base_product.json`。

**用法**：
```bash
python3 full_components_generator.py \
    --subsys=build/subsystem_config.json \
    --out=productdefine/common/base/base_product.json
```

**集成**：扫描所有子系统的 `ohos.build` 和 `bundle.json` 文件。

---

## 5. 组件静态检查工具（`component_tools/static_check/`）

CSCT 是一个全面的静态分析工具，根据 OpenHarmony 编码标准验证组件构建配置。

### 5.1 主入口点

**文件：`csct.py`**

**用途**：本地静态检查 BUILD.gn 和 bundle.json 文件。

**用法**：
```bash
# 检查整个代码库
python3 build/tools/component_tools/static_check/csct.py

# 检查特定路径
python3 build/tools/component_tools/static_check/csct.py -p base/global

# 检查 PR 差异
python3 build/tools/component_tools/static_check/csct.py -cd diff_files.txt
```

**输出**：`out/output_errors.xlsx` - 整合的错误报告

**检查规则**：
- 规则 2.1：组件描述字段必须准确
- 规则 3.1：BUILD.gn 中不能有指向其他组件的绝对/相对路径
- 规则 3.2：所有目标必须指定 `part_name` 和 `subsystem_name`
- 规则 4.1：组件脚本中不能有产品特定变量

### 5.2 在线/CI 版本

**文件**：`csct_online.py`、`csct_online_entry.py`、`csct_online_prehandle.py`

**用途**：CI/CD 集成，用于检查拉取请求。

**流程**：
1. `csct_online_entry.py` - CI 系统的入口点
2. `csct_online_prehandle.py` - 从 Gitee 获取和解析 PR 差异
3. `csct_online.py` - 仅对修改的文件运行检查

**用法**：
```bash
python3 csct_online.py "https://gitee.com/repo/pulls/123"
```

### 5.3 GN 检查模块

**文件**：`gn_check/check_gn.py`、`gn_check/check_gn_online.py`、`gn_check/gn_common_tools.py`

**用途**：根据编码标准验证 BUILD.gn 文件。

**检查项**：
- 绝对路径引用（规则 3.1）
- 缺少 `subsystem_name`/`part_name`（规则 3.2）
- 产品特定代码（规则 4.1）

**关键类**：`CheckGn` - 带有白名单支持的主检查引擎

### 5.4 Bundle 检查模块

**文件**：
- `bundle_check/bundle_json_check.py` - 完整的 bundle.json 验证
- `bundle_check/bundle_check_online.py` - PR 差异检查
- `bundle_check/bundle_check_common.py` - 实用函数
- `bundle_check/get_subsystem_with_component.py` - 子系统映射
- `bundle_check/warning_info.py` - 错误消息常量

**用途**：验证 bundle.json 组件描述符文件。

**验证项**：
- 名称格式：`@organization/component_name`
- 版本与 OpenHarmony 版本匹配
- destPath 是相对且有效的
- 子系统为小写
- syscap 遵循 SystemCapability 命名
- rom/ram 值带有适当的单位

**用法**：
```bash
# 检查所有 bundle.json 文件
python3 bundle_json_check.py -P /path/to/project

# 检查特定文件
python3 bundle_json_check.py -p path/to/bundle.json
```

### 5.5 白名单配置

**文件**：`config/csct_whitelist.conf`

包含排除检查的目录：
- `out` - 构建输出
- `vendor` - 供应商特定代码
- `device` - 设备特定代码
- `third_party` - 外部依赖

---

## 构建系统集成

### GN 构建集成

大多数工具依赖于 GN 构建输出：

1. **启用依赖跟踪**：
   ```bash
   ./build.sh --product-name {product} --gn-args check_deps=true --build-only-gn
   ```

2. **输出位置**：`out/{product}/deps_files/`

3. **文件格式**（每个模块的 JSON）：
   ```json
   {
     "deps": ["//path/to:module"],
     "external_deps": ["part:module"],
     "module_label": "//path:to(//toolchain)",
     "part_name": "part_name"
   }
   ```

### CI/CD 集成

1. **合并前检查**：使用 `csct_online_entry.py` 验证 PR
2. **依赖跟踪**：在夜间构建中使用 `check_deps.py`
3. **文档**：所有工具都支持 `--help` 获取使用信息

---

## 依赖项

### 必需的 Python 包

```bash
# 用于依赖可视化
pip3 install pyecharts

# 用于静态检查
pip3 install prettytable pandas openpyxl

# 用于组件分析
pip3 install graphviz
```

### 系统依赖

- `grep` - GN 检查工具使用
- `find` - 用于文件发现
- `curl` - 在线 PR 检查器使用

---

## 关键脚本摘要

| 脚本 | 用途 | 使用频率 |
|------|------|----------|
| `csct.py` | 静态检查 BUILD.gn 和 bundle.json | 日常开发 |
| `check_deps.py` | 验证 deps 与 external_deps 的使用 | 构建验证 |
| `part_deps.py` | 生成组件依赖图 | 架构分析 |
| `module_deps_tree.py` | 可视化单个模块依赖 | 调试 |
| `full_components_generator.py` | 生成基础产品配置 | 发布准备 |
| `generate_kconfig.py` | 从产品创建 Kconfig | 产品定制 |
| `parse_kconf.py` | 将 .config 转换为 product.json | 产品配置 |
| `product_config_version_convert.py` | 迁移旧配置 | 一次性迁移 |

---

## 注意事项

- 所有工具均使用 Python 3 编写
- 大多数工具支持完整代码库和增量（差异）检查
- 错误消息主要使用中文，以匹配 OpenHarmony 文档
- 工具的编码标准文档位于：https://gitee.com/openharmony/docs/blob/master/zh-cn/device-dev/subsystems/subsys-build-component-building-rules.md

（文件结束 - 共 420 行）
