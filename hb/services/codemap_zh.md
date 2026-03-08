# HarmonyOS 构建系统 (hb) - 服务层代码地图

## 目录

1. [概述](#概述)
2. [服务架构](#服务架构)
3. [接口层次结构](#接口层次结构)
4. [服务实现](#服务实现)
5. [设计模式](#设计模式)
6. [数据与控制流](#数据与控制流)
7. [集成点](#集成点)
8. [文件参考](#文件参考)

---

## 概述

`services/` 目录包含 HarmonyOS 构建系统 (hb) 的核心服务层。该层实现了**服务层模式**和**外观模式**，将复杂的构建操作抽象为内聚的、可管理的单元。每个服务封装特定的构建阶段功能，并与外部工具（GN、Ninja、HPM、HDC）协调。

### 关键职责

- **构建文件生成**：生成 GN 构建文件和配置（`gn.py`、`loader.py`、`preloader.py`）
- **构建执行**：通过 Ninja 执行编译（`ninja.py`）
- **包管理**：处理 HPM（HarmonyOS 包管理器）操作（`hpm.py`、`prebuilts.py`）
- **设备通信**：管理 HDC（HarmonyOS 设备连接器）操作（`hdc.py`）
- **交互式配置**：提供 CLI 菜单界面（`menu.py`）
- **SDK 管理**：处理预构建 SDK 操作（`prebuilt_sdk.py`）
- **独立构建**：支持组件级独立构建（`indep_build.py`）

---

## 服务架构

### 分层架构

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLI / 入口点                                   │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │    菜单     │  │   构建      │  │      预构建SDK          │  │
│  │  (menu.py)  │  │   服务      │  │   (prebuilt_sdk.py)     │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │                │                     │                │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌───────────▼────────────┐   │
│  │   预加载    │  │   加载器    │  │   构建生成器           │   │
│  │(preloader)  │  │  (loader)   │  │  (gn, hpm, hdc)        │   │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬────────────┘   │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │                │                     │                │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌───────────▼────────────┐   │
│  │   构建      │  │  工具层     │  │   外部工具             │   │
│  │  执行器     │  │             │  │ (GN/Ninja/HPM/HDC)     │   │
│  │  (ninja)    │  │             │  │                        │   │
│  └─────────────┘  └─────────────┘  └────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      接口抽象层                                    │
├─────────────────────────────────────────────────────────────────┤
│  ServiceInterface  BuildFileGenerator  BuildExecutor  Load      │
│  PreloadInterface  MenuInterface       PrebuiltSdkInterface     │
└─────────────────────────────────────────────────────────────────┘
```

---

## 接口层次结构

### 基础接口：`ServiceInterface`

**文件**：`services/interface/service_interface.py`

```python
class ServiceInterface(metaclass=ABCMeta):
    - _args_dict: dict      # 参数字典
    - _exec: str            # 可执行文件路径
    + args_dict: property
    + exec: property
    + regist_arg(arg_name: str, arg_value): abstract
    + run(): abstract
```

所有服务的根抽象。定义参数注册和执行的约定。

### 专用接口

#### 1. `BuildFileGeneratorInterface`

**文件**：`services/interface/build_file_generator_interface.py`

```python
class BuildFileGeneratorInterface(ServiceInterface):
    - _flags_dict: dict
    + flags_dict: property
    + regist_flag(flag_name: str, flag_value)
    + run(): abstract
```

**实现类**：`Gn`、`Hpm`、`Hdc`、`IndepBuild`、`PreuiltsService`

用途：生成或操作构建文件和配置的服务。通过标志管理扩展基础接口。

#### 2. `BuildExecutorInterface`

**文件**：`services/interface/build_executor_interface.py`

```python
class BuildExecutorInterface(ServiceInterface):
    - _start_time: timestamp
    + regist_arg(arg_name: str, arg_value: str)
    + run(): abstract
```

**实现类**：`Ninja`

用途：执行构建过程的服务。跟踪执行开始时间以进行度量。

#### 3. `LoadInterface`

**文件**：`services/interface/load_interface.py`

```python
class LoadInterface(ServiceInterface):
    - _config: Config
    + config: property
    + outputs: property
    + __post_init(): abstract
    + run()  # 模板方法
    + _generate_*(): abstract methods
    + _check_*(): abstract methods
```

**实现类**：`OHOSLoader`

用途：用于构建配置加载的模板方法模式。定义 17 个抽象生成方法。

#### 4. `PreloadInterface`

**文件**：`services/interface/preload_interface.py`

```python
class PreloadInterface(ServiceInterface):
    - _config: Config
    + config: property
    + outputs: property
    + __post_init(): abstract
    + run()  # 模板方法
    + _generate_*(): abstract methods
```

**实现类**：`OHOSPreloader`

用途：用于预加载器阶段的模板方法模式。生成预构建配置文件。

#### 5. `MenuInterface`

**文件**：`services/interface/menu_interface.py`

```python
class MenuInterface:
    + select_product(): dict
    + select_compile_option(): dict
```

**实现类**：`Menu`

用途：抽象交互式 CLI 菜单操作。

#### 6. `PrebuiltSdkInterface`

**文件**：`services/interface/prebuilt_sdk_interface.py`

```python
class PrebuiltSdkInterface(ServiceInterface):
    - _config: Config
    + config: property
    + should_build_sdk(args_dict): bool
    + build_prebuilt_sdk(args_dict): bool
    + _execute_sdk_build(build_args: dict): bool
    + _post_process_sdk(api_version: str): bool
    + _migrate_legacy_sdk(): None
```

**实现类**：`PrebuiltSdk`

用途：管理 SDK 构建和生命周期操作。

---

## 服务实现

### 1. GN 服务（`gn.py`）

**类**：`Gn(BuildFileGeneratorInterface)`

**职责**：GN（生成 Ninja）元构建系统的接口。

**关键组件**：

```python
class CMDTYPE(Enum):
    GEN = 1      # 生成构建文件
    PATH = 2     # 显示目标之间的路径
    DESC = 3     # 描述目标
    LS = 4       # 列出目标
    REFS = 5     # 引用追踪
    FORMAT = 6   # 格式化 .gn 文件
    CLEAN = 7    # 清理构建目录
```

**关键方法**：
- `_regist_gn_path()`：定位 GN 可执行文件（平台感知：x86/aarch64）
- `_execute_gn_gen_cmd()`：带线程动画的核心 GN 生成
- `_convert_args()`：将 args_dict 转换为 GN 兼容格式
- `_convert_flags()`：将 flags_dict 转换为 GN 标志
- `_check_options_validity()`：验证命令选项

**设计模式**：**命令模式** - 每种 GN 命令类型封装为通过 `execute_gn_cmd()` 调度器执行的方法。

**线程**：在静默模式下 GN 解析期间使用 `threading.Event` 和 `threading.Thread` 实现动画加载旋转器。

---

### 2. Ninja 服务（`ninja.py`）

**类**：`Ninja(BuildExecutorInterface)`

**职责**：使用 Ninja 执行实际构建编译。

**关键方法**：
- `_regist_ninja_path()`：定位 Ninja 可执行文件
- `_execute_ninja_cmd()`：执行带环境过滤的构建
- `_convert_args()`：将参数转换为 Ninja 命令格式

**环境管理**：
```python
ninja_env = ExecEnviron()
ninja_env.initenv()
ninja_env.allow(ninja_env_allowlist)  # 基于白名单的环境过滤
```

**安全特性**：使用 `compile_env_allowlist.json` 限制传递给 Ninja 的环境变量。

---

### 3. HPM 服务（`hpm.py`）

**类**：`Hpm(BuildFileGeneratorInterface)`

**职责**：HarmonyOS 包管理器集成，用于二进制依赖管理。

**命令类型**：
```python
class CMDTYPE(Enum):
    BUILD = 1
    INSTALL = 2
    PACKAGE = 3
    PUBLISH = 4
    UPDATE = 5
```

**关键特性**：
- **版本检查**：后台线程通过 npm 注册表检查 HPM 更新
- **进度处理**：提取进度的自定义行处理程序（`_custom_line_handle()`）
- **跳过逻辑**：支持 `--skip-download`、`--fast-rebuild`、`--local-binarys` 标志

**重试机制**：HPM 构建操作 `max_try=3`。

---

### 4. HDC 服务（`hdc.py`）

**类**：`Hdc(BuildFileGeneratorInterface)`

**职责**：HarmonyOS 设备连接器，用于设备部署。

**命令类型**：
```python
class CMDTYPE(Enum):
    PUSH = 1           # 推送文件到设备
    LIST_TARGETS = 2   # 列出已连接设备
```

**部署流程**：
1. 将文件系统挂载为可读写
2. 解析组件 bundle.json 以获取部署配置
3. 通过 HDC 发送文件
4. 设置文件所有权（root:root）
5. 可选设备重启

**Bundle 集成**：读取 `bundle.json` 的 `deployment` 部分以获取源/目标路径。

---

### 5. 加载器服务（`loader.py`）

**类**：`OHOSLoader(LoadInterface)`

**职责**：用于加载和生成构建配置的模板方法实现。

**模板方法执行**（继承的 `run()`）：
```python
1. __post_init()              # 初始化配置
2. _execute_loader_args_display()
3. _check_parts_config_info()
4. _generate_subsystem_configs()
5. _generate_target_platform_parts()
6. _generate_system_capabilities()
7. _generate_stub_targets()
8. _generate_platforms_part_by_src()
9. _generate_target_gn()
10. _generate_phony_targets_build_file()
11. _generate_required_parts_targets()
12. _generate_required_parts_targets_list()
13. _generate_src_flag()
14. _generate_auto_install_part()
15. _generate_platforms_list()
16. _generate_part_different_info()
17. _generate_infos_for_testfwk()
18. _check_product_part_feature()
19. _generate_syscap_files()
20. _cropping_components()
```

**关键能力**：
- 子系统配置解析
- 平台特定部件变体解析
- 组件覆盖机制（`_override_one_component()`）
- 系统能力（syscap）文件生成
- 组件分发处理（`_load_component_dist()`）

**验证**：
- 针对白名单的产品部件特性验证
- 部件配置完整性检查

---

### 6. 预加载器服务（`preloader.py`）

**类**：`OHOSPreloader(PreloadInterface)`

**职责**：预构建配置生成（模板方法模式）。

**生成的产物**：
| 文件 | 用途 |
|------|------|
| `platforms.build` | 平台构建配置 |
| `build_gnargs.prop` | GN 参数属性 |
| `features.json` | 部件特性映射 |
| `syscap.json` | 系统能力 |
| `exclusion_modules.json` | 排除的模块 |
| `build_config.json` | 构建变量 |
| `build.prop` | 构建属性 |
| `parts.json` | 产品部件列表 |
| `parts_config.json` | 部件布尔配置 |
| `subsystem_config.json` | 子系统定义 |
| `systemcapability.json` | 系统能力信息 |
| `compile_standard_whitelist.json` | 编译标准白名单 |
| `compile_env_allowlist.json` | 环境白名单 |
| `hvigor_compile_hap_whitelist.json` | HAP 白名单 |

**操作系统级别支持**：
- `standard`：完整子系统配置
- `mini`/`small`：通过 `parse_lite_subsystem_config()` 实现精简子系统配置

---

### 7. 菜单服务（`menu.py`）

**类**：`Menu(MenuInterface)`

**职责**：使用 `prompt_toolkit` 的交互式 CLI 菜单。

**组件**：
- `InquirerControl`：用于选择 UI 的自定义令牌列表控件
- `_list_promt()`：基于列表的提示处理程序
- `_question()`：带键绑定的应用程序组装

**键绑定**：
- `Ctrl+Q`/`Ctrl+C`：取消
- `上`/`下`：导航（跳过分隔符/禁用项）
- `回车`：选择确认

**选择流程**：
1. 选择 OS 级别（mini/small/standard）
2. 选择产品（按公司分组）
3. 按参数选择编译选项

---

### 8. 预构建 SDK 服务（`prebuilt_sdk.py`）

**类**：`PrebuiltSdk(PrebuiltSdkInterface)`

**职责**：SDK 构建和管理。

**构建决策逻辑**：
```python
should_build_sdk():
  - 如果 --no-prebuilt-sdk=true 则跳过
  - 如果产品是 'ohos-sdk' 则跳过
  - 如果 SDK 已存在则跳过
  - 如果 --prebuilt-sdk=false 则跳过
```

**构建流水线**：
1. `_set_path()`：为 ccache 和 Node.js 配置 PATH
2. `_execute_sdk_build()`：使用特定 GN 参数构建 SDK 产品
3. `_post_process_sdk()`：组织输出结构
4. `_create_previewer_package()`：创建预览器变体
5. `_migrate_legacy_sdk()`：迁移旧版 SDK-12

**SDK 的 GN 参数**：
```python
[
    'skip_generate_module_list_file=true',
    'sdk_platform={platform}',
    'use_cfi=false',
    'use_thin_lto=false',
    'enable_lto_O0=true',
    'sdk_check_flag=false',
    'sdk_for_hap_build=true',
    ...
]
```

---

### 9. 预构建服务（`prebuilts.py`）

**类**：`PreuiltsService(BuildFileGeneratorInterface)`

**职责**：预构建二进制依赖管理。

**增量更新逻辑**：
```python
check_whether_need_update():
  - 无先前更新 → 更新
  - 部件名称更改 → 更新
  - 配置文件比上次更新新 → 更新
```

**跟踪的文件**：
- `build/prebuilts_service/*`
- `build/prebuilts_config.json`
- `build/prebuilts_config.py`
- `build/prebuilts_config.sh`

**状态持久化**：`prebuilts/.local_data/last_update.json`

---

### 10. 独立构建服务（`indep_build.py`）

**类**：`IndepBuild(BuildFileGeneratorInterface)`

**职责**：组件级独立构建。

**构建脚本**：执行 `build/indep_configs/build_indep.sh`

**特性**：
- 本地二进制缓存支持（`--local-binarys`）
- 依赖 JSON 生成（`_generate_dependences_json()`）
- 构建类型变体：`both`、`onlytest`、`normal`

**依赖管理**：
- 为本地二进制文件创建符号链接
- 生成 `dependences.json` 用于组件映射

---

## 设计模式

### 1. 服务层模式

**实现**：所有服务实现 `ServiceInterface` 或派生接口。

**优点**：
- 关注点清晰分离
- 一致的参数注册 API
- 可插拔服务实现
- 通过接口模拟实现可测试性

### 2. 模板方法模式

**实现**：`LoadInterface.run()` 和 `PreloadInterface.run()`

```python
class LoadInterface:
    def run(self):
        self.__post_init__()
        self._generate_subsystem_configs()
        self._generate_target_platform_parts()
        # ... 更多步骤
```

**优点**：
- 固定执行序列
- 可定制的单个步骤
- 跨变体的一致加载器行为

### 3. 命令模式

**实现**：`Gn.execute_gn_cmd()`、`Hpm.execute_hpm_cmd()`、`Hdc.execute_hdc_cmd()`

```python
def execute_gn_cmd(self, cmd_type: int, **kwargs):
    if cmd_type == CMDTYPE.GEN:
        return self._execute_gn_gen_cmd()
    elif cmd_type == CMDTYPE.PATH:
        return self._execute_gn_path_cmd(**kwargs)
    # ...
```

**优点**：
- 封装命令执行
- 支持每个命令的参数变化
- 可扩展以支持新命令类型

### 4. 外观模式

**实现**：服务为复杂子系统提供简化接口。

示例：
- `Gn` 抽象 GN 工具复杂性
- `Ninja` 抽象构建执行
- `Loader` 协调多个工具模块

### 5. 策略模式

**实现**：平台特定的可执行文件解析。

```python
if sys.platform == "linux" and platform.machine().lower() == "aarch64":
    gn_path = ".../linux-aarch64/bin/gn"
else:
    gn_path = ".../linux-x86/bin/gn"
```

---

## 数据与控制流

### 构建流水线流程

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   预加载器   │────▶│    加载器    │────▶│      GN      │
│  (配置)      │     │   (生成)     │     │   (元)       │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    HDC       │◀────│    Ninja     │◀────│    HPM       │
│  (部署)      │     │   (编译)     │     │  (二进制)    │
└──────────────┘     └──────────────┘     └──────────────┘
```

### 预加载器数据流

```
产品配置 ──┐
子系统信息 ──┼──▶ OHOSPreloader ──▶ JSON/PROP 文件 ──▶ 加载器输入
构建变量 ────┘    (__post_init__)    (out/preloader/)
```

### 加载器数据流

```
子系统配置 ──┐
平台配置 ────┼──▶ OHOSLoader ──┬──▶ GN 构建文件
部件信息 ────┘   (__post_init__)  ├──▶ JSON 配置
                                  └──▶ 目标列表
```

### 服务执行流程

```
1. CLI 解析参数
2. Arg.register_args() 填充服务
3. Service.run() 执行
   a. __post_init__() - 初始化状态
   b. 执行生成/执行方法
   c. 记录结果
4. 通过 @throw_exception 进行异常处理
```

---

## 集成点

### 1. 工具层依赖

| 服务 | 工具依赖 |
|------|---------|
| `Gn` | `SystemUtil`、`IoUtil`、`LogUtil` |
| `Ninja` | `SystemUtil`、`IoUtil`、`LogUtil` |
| `Hpm` | `SystemUtil`、`ComponentUtil`、`LogUtil` |
| `Hdc` | `SystemUtil`、`IoUtil`、`ComponentUtil` |
| `Loader` | `loader.*`、`file_utils`、`LogUtil` |
| `Preloader` | `IoUtil`、`preloader_process_data`、`LogUtil` |

### 2. 资源依赖

| 服务 | 资源依赖 |
|------|----------|
| 全部 | `Config`、`Arg` |
| `PrebuiltSdk` | `CURRENT_OHOS_ROOT`、`global_var` |
| `Hpm` | `CURRENT_OHOS_ROOT`、`global_var` |
| `PreuiltsService` | `CURRENT_OHOS_ROOT` |

### 3. 外部工具集成

| 服务 | 外部工具 | 版本/路径解析 |
|------|----------|--------------|
| `Gn` | GN | `prebuilts/build-tools/{platform}-{arch}/bin/gn` |
| `Ninja` | Ninja | `prebuilts/build-tools/{platform}-{arch}/bin/ninja` |
| `Hpm` | HPM | `shutil.which("hpm")` 或 `prebuilts/hpm/node_modules/.bin/hpm` |
| `Hdc` | HDC | `shutil.which("hdc")` |

### 4. 配置文件

| 服务 | 输入配置文件 | 输出文件 |
|------|-------------|----------|
| `Preloader` | `product_config.json`、`subsystem_config.json` | `out/preloader/{product}/*.json` |
| `Loader` | `out/preloader/{product}/*.json` | `out/{product}/build_configs/*.json` |
| `Gn` | `out/{product}/args.gn` | `out/{product}/build.ninja` |
| `PreuiltsService` | `build/prebuilts_config.json` | `prebuilts/.local_data/last_update.json` |

### 5. 模块依赖

```
services/
├── gn.py ───────────┬──▶ util/system_util
├── ninja.py ────────┤    util/io_util
├── hpm.py ──────────┤    util/log_util
├── hdc.py ──────────┤    util/component_util
├── loader.py ───────┼──▶ util/loader/*
├── preloader.py ────┤    util/preloader/*
├── menu.py ─────────┤    resources/config
├── prebuilt_sdk.py ─┤    resources/global_var
├── prebuilts.py ────┤    containers/arg
└── indep_build.py ──┘    exceptions/ohos_exception
```

---

## 文件参考

### 接口文件

| 文件 | 行数 | 用途 |
|------|------|------|
| `interface/service_interface.py` | 45 | 基础服务抽象 |
| `interface/build_file_generator_interface.py` | 42 | 构建文件生成约定 |
| `interface/build_executor_interface.py` | 41 | 构建执行约定 |
| `interface/load_interface.py` | 145 | 加载器模板方法 |
| `interface/preload_interface.py` | 120 | 预加载器模板方法 |
| `interface/menu_interface.py` | 30 | 菜单抽象 |
| `interface/prebuilt_sdk_interface.py` | 55 | SDK 管理约定 |

### 实现文件

| 文件 | 行数 | 类 | 接口 | 主要功能 |
|------|------|------|------|----------|
| `gn.py` | 329 | `Gn` | `BuildFileGeneratorInterface` | GN 元构建编排 |
| `ninja.py` | 108 | `Ninja` | `BuildExecutorInterface` | 构建执行 |
| `hpm.py` | 263 | `Hpm` | `BuildFileGeneratorInterface` | 包管理 |
| `hdc.py` | 138 | `Hdc` | `BuildFileGeneratorInterface` | 设备部署 |
| `loader.py` | 996 | `OHOSLoader` | `LoadInterface` | 构建配置生成 |
| `preloader.py` | 348 | `OHOSPreloader` | `PreloadInterface` | 预构建配置 |
| `menu.py` | 423 | `Menu` | `MenuInterface` | 交互式 CLI |
| `prebuilt_sdk.py` | 305 | `PrebuiltSdk` | `PrebuiltSdkInterface` | SDK 构建 |
| `prebuilts.py` | 161 | `PreuiltsService` | `BuildFileGeneratorInterface` | 二进制依赖管理 |
| `indep_build.py` | 152 | `IndepBuild` | `BuildFileGeneratorInterface` | 组件构建 |

### 统计总计

- **代码总行数**：约 2,891（仅服务）
- **接口文件**：7
- **服务实现**：10
- **使用的设计模式**：5+

---

## 异常处理

所有服务使用 `containers.status` 中的 `@throw_exception` 装饰器进行一致的错误处理：

```python
@throw_exception
def _execute_gn_gen_cmd(self, **kwargs):
    # 失败时抛出 OHOSException
```

错误代码已标准化（例如，'0001' 表示缺少可执行文件，'3001' 表示不支持的命令类型）。

---

## 结论

服务层实现了结构良好、接口驱动的架构，抽象了 HarmonyOS 构建操作的复杂性。设计模式（服务层、模板方法、命令、外观）的使用确保了可维护性、可测试性和可扩展性。构建文件生成（`Gn`、`Hpm`、`Loader`、`Preloader`）、构建执行（`Ninja`）和工具服务（`Hdc`、`Menu`、`PrebuiltSdk`）之间的清晰分离实现了模块化开发和调试。

（文件结束 - 共 702 行）
