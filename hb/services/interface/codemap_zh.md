# services/interface/ 代码地图

## 概述

`services/interface/` 目录定义了鸿蒙构建系统（hb）的**抽象服务层**。该模块建立了服务契约和抽象基类（ABC），用于规范构建系统编排逻辑与其具体实现之间的架构边界。

---

## 1. 职责

此目录包含**服务接口定义**——纯抽象类，定义了构建系统中所有主要服务组件的行为契约。每个接口在构建流水线中确立了特定的领域边界。

### 1.1 核心服务分类

| 接口 | 领域 | 用途 |
|-----------|--------|---------|
| `ServiceInterface` | 基础 | 所有服务的根抽象 |
| `BuildExecutorInterface` | 执行 | 构建执行引擎（Ninja）的契约 |
| `BuildFileGeneratorInterface` | 生成 | 构建文件生成器（GN）的契约 |
| `LoadInterface` | 加载 | 子系统/部件配置加载的契约 |
| `PreloadInterface` | 预加载 | 产品预加载阶段的契约 |
| `PrebuiltSdkInterface` | SDK | 预构建SDK管理的契约 |
| `MenuInterface` | UI | 交互式CLI菜单的契约 |

### 1.2 架构角色

```
┌─────────────────────────────────────────────────────────────────┐
│                    服务接口层                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐   │
│  │   Service   │ │   Build     │ │   BuildFileGenerator    │   │
│  │  Interface  │ │  Executor   │ │      Interface          │   │
│  │   (Base)    │ │  Interface  │ │                         │   │
│  └──────┬──────┘ └──────┬──────┘ └───────────┬─────────────┘   │
│         │               │                    │                 │
│  ┌──────┴──────┐ ┌──────┴──────┐  ┌──────────┴─────────────┐   │
│  │    Load     │ │   Preload   │  │   PrebuiltSdkInterface │   │
│  │  Interface  │ │  Interface  │  │                        │   │
│  └─────────────┘ └─────────────┘  └────────────────────────┘   │
│                                                                │
│  ┌─────────────────┐                                           │
│  │  MenuInterface  │  （独立 - 无继承）            │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. 设计模式

### 2.1 抽象基类（ABC）模式

所有接口都使用 Python 的 `abc` 模块，配合 `ABCMeta` 元类和 `@abstractmethod` 装饰器来强制实施实现契约。

```python
# 模式：带模板方法的抽象基类
class ServiceInterface(metaclass=ABCMeta):
    @abstractmethod
    def run(self):
        """模板方法 - 必须由子类实现"""
        pass
```

### 2.2 模板方法模式

`LoadInterface` 和 `PreloadInterface` 实现了**模板方法模式**，定义固定的执行顺序，同时将具体步骤委托给具体实现：

```python
# LoadInterface.run() - 模板方法
def run(self):
    self.__post_init__()                           # 钩子
    self._execute_loader_args_display()           # 步骤 1
    self._check_parts_config_info()               # 步骤 2
    self._generate_subsystem_configs()            # 步骤 3
    # ... 还有 18 个步骤按固定顺序执行
    self._cropping_components()                   # 最后一步
```

### 2.3 策略模式

`BuildFileGeneratorInterface` 为不同的构建生成后端启用**策略模式**：
- `Gn` - GN 元构建系统
- `IndepBuild` - 独立构建系统
- `PreuiltsService` - 预构建下载服务

### 2.4 接口隔离

每个接口都遵循**接口隔离原则**，具有聚焦的职责：

| 接口 | 方法数量 | 内聚性 |
|-----------|-------------|----------|
| `ServiceInterface` | 2 | 核心生命周期（regist_arg, run） |
| `MenuInterface` | 2 | UI 选择（select_product, select_compile_option） |
| `BuildExecutorInterface` | 1 | 执行（run） |
| `BuildFileGeneratorInterface` | 1 | 生成（run） |
| `PrebuiltSdkInterface` | 5 | SDK 生命周期管理 |

### 2.5 继承层次结构

```
ServiceInterface (ABCMeta)
├── BuildExecutorInterface
│   └── Ninja
├── BuildFileGeneratorInterface
│   ├── Gn
│   ├── IndepBuild
│   └── PreuiltsService
├── LoadInterface
│   └── OHOSLoader
├── PreloadInterface
│   └── OHOSPreloader
└── PrebuiltSdkInterface
    └── PrebuiltSdk

MenuInterface (Standalone ABC)
└── Menu
```

---

## 3. 数据与控制流

### 3.1 服务契约规范

#### 3.1.1 ServiceInterface（根契约）

**位置**：`service_interface.py:21`

```python
class ServiceInterface(metaclass=ABCMeta):
    def __init__(self):
        self._args_dict = {}    # 运行时参数注册表
        self._exec = ''         # 可执行文件路径

    @abstractmethod
    def regist_arg(self, arg_name: str, arg_value):
        """注册构建参数 - 实现特定"""
        pass

    @abstractmethod
    def run(self):
        """执行服务 - 主入口点"""
        pass
```

**数据流**：
```
参数解析 → regist_arg() → _args_dict → run() → 执行
```

#### 3.1.2 BuildExecutorInterface（执行契约）

**位置**：`build_executor_interface.py:25`

| 属性 | 类型 | 用途 |
|-----------|------|---------|
| `_start_time` | int | 执行计时（Unix 时间戳） |
| `_args_dict` | dict | 继承自 ServiceInterface |
| `_exec` | str | 可执行文件路径（ninja） |

**控制流**：
```
__init__() → SystemUtil.get_current_time() → _start_time
regist_arg() → 重复检查 → _args_dict[key] = value
run() → 重置 _start_time → 执行构建
```

#### 3.1.3 BuildFileGeneratorInterface（生成契约）

**位置**：`build_file_generator_interface.py:24`

| 属性 | 类型 | 用途 |
|-----------|------|---------|
| `_flags_dict` | dict | 构建标志注册表 |
| `_args_dict` | dict | 继承的参数 |

**关键方法**：
- `regist_flag(flag_name, flag_value)` - 注册构建标志
- `regist_arg(arg_name, arg_value)` - 注册参数（直接赋值，无重复检查）

#### 3.1.4 LoadInterface（加载契约）

**位置**：`load_interface.py:24`

**状态管理**：
| 属性 | 类型 | 来源 |
|-----------|------|--------|
| `_config` | Config | resources.config.Config 单例 |
| `_outputs` | Any | 实现定义输出 |
| `_args_dict` | dict | 继承的参数注册表 |

**模板方法序列**（20 个步骤）：

```python
def run(self):
    self.__post_init__()                    # 0. 初始化钩子
    self._execute_loader_args_display()     # 1. 日志记录
    self._check_parts_config_info()         # 2. 验证
    self._generate_subsystem_configs()      # 3. 子系统配置
    self._generate_target_platform_parts()  # 4. 平台部件
    self._generate_system_capabilities()    # 5. 系统能力
    self._generate_stub_targets()           # 6. 桩目标
    self._generate_platforms_part_by_src()  # 7. 按源平台部件
    self._generate_target_gn()              # 8. GN 目标文件
    self._generate_phony_targets_build_file()  # 9. 伪目标
    self._generate_required_parts_targets()    # 10. 必需部件
    self._generate_required_parts_targets_list()  # 11. 部件列表
    self._generate_src_flag()               # 12. 源标志
    self._generate_auto_install_part()      # 13. 自动安装
    self._generate_platforms_list()         # 14. 平台列表
    self._generate_part_different_info()    # 15. 部件差异
    self._generate_infos_for_testfwk()      # 16. 测试框架信息
    self._check_product_part_feature()      # 17. 特性验证
    self._generate_syscap_files()           # 18. Syscap 文件
    self._cropping_components()             # 19. 组件裁剪
```

#### 3.1.5 PreloadInterface（预加载契约）

**位置**：`preload_interface.py:24`

**状态管理**：
| 属性 | 类型 | 用途 |
|-----------|------|---------|
| `_config` | Config | 产品配置 |
| `_preloader_outputs` | Any | 生成的输出产物 |

**模板方法序列**（15 个步骤）：

```python
def run(self):
    self.__post_init__()                           # 0. 初始化
    self._generate_build_prop()                    # 1. 构建属性
    self._generate_build_config_json()             # 2. 构建配置
    self._generate_parts_json()                    # 3. 部件列表
    self._generate_parts_config_json()             # 4. 部件配置
    self._generate_build_gnargs_prop()             # 5. GN 参数
    self._generate_features_json()                 # 6. 特性
    self._generate_syscap_json()                   # 7. 系统能力
    self._generate_exclusion_modules_json()        # 8. 排除项
    self._generate_platforms_build()               # 9. 平台构建配置
    self._generate_subsystem_config_json()         # 10. 子系统配置
    self._generate_systemcapability_json()         # 11. 系统能力
    self._generate_compile_standard_whitelist_json()  # 12. 编译白名单
    self._generate_compile_env_allowlist_json()    # 13. 环境允许列表
    self._generate_hvigor_compile_whitelist_json() # 14. Hvigor 白名单
```

#### 3.1.6 PrebuiltSdkInterface（SDK 契约）

**位置**：`prebuilt_sdk_interface.py:24`

**生命周期方法**：
| 方法 | 返回值 | 用途 |
|--------|--------|---------|
| `should_build_sdk(args_dict)` | bool | SDK 构建必要性谓词 |
| `build_prebuilt_sdk(args_dict)` | bool | 主 SDK 构建编排 |
| `_execute_sdk_build(build_args)` | bool | 执行 SDK 编译 |
| `_post_process_sdk(api_version)` | bool | 构建后产物处理 |
| `_migrate_legacy_sdk()` | None | 旧版 SDK 迁移 |

#### 3.1.7 MenuInterface（UI 契约）

**位置**：`menu_interface.py:22`

**交互方法**：
| 方法 | 返回值 | 用途 |
|--------|--------|---------|
| `select_product()` | dict | 交互式产品选择 |
| `select_compile_option()` | dict | 交互式构建选项选择 |

---

### 3.2 数据转换流

```
┌────────────────────────────────────────────────────────────────────────┐
│                        构建流水线流                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. 预加载阶段（PreloadInterface）                                   │
│     ┌──────────────┐                                                   │
│     │ OHOSPreloader│ → 生成：build.prop, parts.json,             │
│     │              │              features.json, syscap.json, ...      │
│     └──────┬───────┘     输出：out/preloader/${product}/            │
│            │                                                           │
│  2. 加载阶段（LoadInterface）                                         │
│     ┌──────────────┐                                                   │
│     │  OHOSLoader  │ → 消费：Preloader 输出                    │
│     │              │ → 生成：BUILD.gn, subsystem configs,        │
│     └──────┬───────┘              parts_targets, system_capabilities   │
│            │                   输出：out/${product}/build_configs/   │
│            │                                                           │
│  3. 生成阶段（BuildFileGeneratorInterface）                     │
│     ┌──────────┐  ┌──────────────┐  ┌──────────────┐                  │
│     │    Gn    │  │  IndepBuild  │  │PreuiltsService│                  │
│     │          │  │              │  │               │                  │
│     └────┬─────┘  └──────┬───────┘  └───────┬───────┘                  │
│          │               │                  │                         │
│          └───────────────┴──────────────────┘                         │
│                          │                                            │
│  4. 执行阶段（BuildExecutorInterface）                           │
│     ┌──────────┐                                                       │
│     │  Ninja   │ → 消费：build.ninja（来自 GN）                    │
│     │          │ → 执行：并行构建执行                 │
│     └──────────┘                                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. 集成点

### 4.1 具体实现

| 接口 | 实现 | 文件 | 职责 |
|-----------|---------------|------|----------------|
| `BuildExecutorInterface` | `Ninja` | `services/ninja.py:31` | 并行构建执行 |
| `BuildFileGeneratorInterface` | `Gn` | `services/gn.py:47` | GN 元构建生成 |
| `BuildFileGeneratorInterface` | `IndepBuild` | `services/indep_build.py:30` | 独立组件构建 |
| `BuildFileGeneratorInterface` | `PreuiltsService` | `services/prebuilts.py:32` | 预构建二进制下载 |
| `LoadInterface` | `OHOSLoader` | `services/loader.py:35` | 子系统/部件加载 |
| `PreloadInterface` | `OHOSPreloader` | `services/preloader.py:28` | 产品预加载 |
| `PrebuiltSdkInterface` | `PrebuiltSdk` | `services/prebuilt_sdk.py:31` | SDK 构建管理 |
| `MenuInterface` | `Menu` | `services/menu.py:45` | 交互式 CLI |

### 4.2 实现继承细节

#### 4.2.1 Ninja（构建执行器）

```python
class Ninja(BuildExecutorInterface):
    def __init__(self):
        super().__init__()           # 设置 _start_time
        self._regist_ninja_path()    # 定位 ninja 可执行文件
    
    def run(self):
        self._execute_ninja_cmd()    # 重写 - 执行 ninja
    
    def _convert_args(self) -> list:
        # 将 _args_dict 转换为 ninja CLI 参数
```

**集成点**：
- 消费：`Config.out_path`（输出目录）
- 消费：`args_dict['build_target']`（目标）
- 消费：`args_dict['ninja_args']`（额外参数）
- 使用：`ExecEnviron` 用于沙箱环境

#### 4.2.2 Gn（构建文件生成器）

```python
class Gn(BuildFileGeneratorInterface):
    def __init__(self):
        super().__init__()           # 初始化 _flags_dict
        self._regist_gn_path()       # 定位 gn 可执行文件
    
    def run(self):
        self.execute_gn_cmd(CMDTYPE.GEN)
    
    def _convert_args(self) -> list:
        # 将 _args_dict 转换为 --args="..." 格式
    
    def _convert_flags(self) -> list:
        # 将 _flags_dict 转换为 CLI 标志
```

**集成点**：
- 消费：`Config.out_path`, `Config.os_level`
- 消费：`_args_dict`（转换为 `--args=`）
- 消费：`_flags_dict`（转换为 CLI 标志）
- 生成：输出目录中的 `build.ninja`

#### 4.2.3 OHOSLoader（配置加载器）

```python
class OHOSLoader(LoadInterface):
    def __post_init__(self):
        # 从 Config 初始化路径
        # 加载平台信息
        # 加载部件配置信息
    
    def _generate_target_gn(self):
        # 调用 generate_targets_gn.gen_targets_gn()
```

**集成点**：
- 读取：`out/preloader/${product}/subsystem_config.json`
- 读取：`out/preloader/${product}/platforms.build`
- 写入：`out/${product}/build_configs/`
- 使用：`util/loader/` 模块进行解析

#### 4.2.4 OHOSPreloader（产品预加载器）

```python
class OHOSPreloader(PreloadInterface):
    def __post_init__(self):
        self._dirs = Dirs(self._config)
        self._outputs = Outputs(self._dirs.preloader_output_dir)
        self._product = Product(self._dirs, self._config)
```

**集成点**：
- 使用：`util/preloader/preloader_process_data.py`
- 读取：产品配置
- 写入：`out/preloader/${product}/`

### 4.3 外部依赖

| 接口 | 外部模块 | 用途 |
|-----------|-----------------|---------|
| All | `resources.config.Config` | 全局配置单例 |
| All | `util.log_util.LogUtil` | 结构化日志 |
| `BuildExecutorInterface` | `util.system_util.SystemUtil` | 命令执行 |
| `LoadInterface` | `util.loader.*` | 子系统/部件解析 |
| `PreloadInterface` | `util.preloader.*` | 产品数据处理 |
| `PrebuiltSdkInterface` | `resources.global_var.CURRENT_OHOS_ROOT` | 仓库根目录 |

### 4.4 文件输出契约

#### LoadInterface 输出（到 `out/${product}/build_configs/`）

| 方法 | 输出文件 | 格式 |
|--------|-------------|--------|
| `_generate_target_platform_parts` | `target_platforms_parts.json` | JSON |
| `_generate_part_different_info` | `parts_different_info.json` | JSON |
| `_generate_platforms_list` | `platforms_list.gni` | GNI |
| `_generate_src_flag` | `parts_src_flag.json` | JSON |
| `_generate_auto_install_part` | `auto_install_parts.json` | JSON |
| `_generate_required_parts_targets` | `required_parts_targets.json` | JSON |
| `_generate_required_parts_targets_list` | `required_parts_targets_list.json` | JSON |
| `_generate_platforms_part_by_src` | `platforms_parts_by_src.json` | JSON |
| `_generate_target_gn` | `subsystem_info/*.gni` | GNI |
| `_generate_phony_targets_build_file` | `phony_target/BUILD.gn` | GN |
| `_generate_stub_targets` | `${platform}-stub/BUILD.gn` | GN |
| `_generate_system_capabilities` | `${platform}_system_capabilities.json` | JSON |
| `_generate_subsystem_configs` | `subsystem_info/*.json` | JSON |
| `_generate_infos_for_testfwk` | `infos_for_testfwk.json` | JSON |
| `_generate_syscap_files` | `system/etc/SystemCapability.json` | JSON |

#### PreloadInterface 输出（到 `out/preloader/${product}/`）

| 方法 | 输出文件 | 格式 |
|--------|-------------|--------|
| `_generate_build_prop` | `build.prop` | Properties |
| `_generate_build_config_json` | `build_config.json` | JSON |
| `_generate_parts_json` | `parts.json` | JSON |
| `_generate_parts_config_json` | `parts_config.json` | JSON |
| `_generate_build_gnargs_prop` | `build_gnargs.prop` | Properties |
| `_generate_features_json` | `features.json` | JSON |
| `_generate_syscap_json` | `syscap.json` | JSON |
| `_generate_exclusion_modules_json` | `exclusion_modules.json` | JSON |
| `_generate_platforms_build` | `platforms.build` | JSON |
| `_generate_subsystem_config_json` | `subsystem_config.json` | JSON |
| `_generate_systemcapability_json` | `systemcapability.json` | JSON |
| `_generate_compile_standard_whitelist_json` | `compile_standard_whitelist.json` | JSON |
| `_generate_compile_env_allowlist_json` | `compile_env_allowlist.json` | JSON |
| `_generate_hvigor_compile_whitelist_json` | `hvigor_compile_hap_whitelist.json` | JSON |

---

## 5. 使用模式

### 5.1 服务实例化

```python
# 模式 1：直接实例化
from services.ninja import Ninja
executor = Ninja()
executor.regist_arg('build_target', ['target1', 'target2'])
executor.run()

# 模式 2：基于工厂（隐式）
from services.gn import Gn
generator = Gn()
generator.regist_arg('product_name', 'product')
generator.regist_flag('gn_flags', ['--ide=json'])
generator.run()
```

### 5.2 参数注册流

```python
# 带警告的重复检测（BuildExecutorInterface, LoadInterface, PreloadInterface）
def regist_arg(self, arg_name: str, arg_value: str):
    if arg_name in self._args_dict.keys() and self._args_dict[arg_name] != arg_value:
        LogUtil.hb_warning('duplicated regist arg {}, the original value "{}" will be replace to "{}"'.format(
            arg_name, self._args_dict[arg_name], arg_value))
    self._args_dict[arg_name] = arg_value

# 直接赋值（BuildFileGeneratorInterface）
def regist_arg(self, arg_name: str, arg_value: str):
    self._args_dict[arg_name] = arg_value
```

---

## 6. 总结

`services/interface/` 模块通过抽象基类建立了**清晰的关注点分离**，实现了：

1. **可插拔架构**：可以添加新的构建执行器、生成器或加载器，而无需修改编排逻辑
2. **可测试性**：接口支持单元测试的模拟
3. **一致性**：模板方法确保实现间的统一执行顺序
4. **类型安全**：抽象方法在类定义时强制实现完整性

接口层次结构直接映射到构建流水线阶段：**预加载 → 加载 → 生成 → 执行**，每个阶段由一个具体实现履行的专注契约表示。

（文件结束 - 共 508 行）
