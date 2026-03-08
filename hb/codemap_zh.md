# hb/ - 鸿蒙构建系统 (hb)

鸿蒙命令行构建系统，采用分层架构实现，包含命令模式、模板方法模式和依赖注入。支持通过 HPM（鸿蒙包管理器）进行完整源码构建和独立组件构建。

## 职责

**主要用途：** 鸿蒙构建系统的 CLI 入口点和协调层。

**关键职责：**
- **入口点 (`hb/__main__.py`)**：引导程序，验证 OHOS 工作空间并动态加载 `main.py`
- **协调器 (`main.py`)**：中央命令调度器和模块初始化器
- **模块工厂**：创建和配置 11 个命令模块（build、set、clean、env、tool、indep_build、install、package、publish、update、push）
- **依赖注入容器**：将服务、解析器和模块连接在一起
- **路径管理**：为预构建工具（HPM、Node.js）设置 PATH

## 架构概览

### 分层架构

```
┌─────────────────────────────────────────────────────────────┐
│  CLI 入口 (hb/__main__.py)                                   │
├─────────────────────────────────────────────────────────────┤
│  协调器 (main.py) - 命令调度                                 │
├─────────────────────────────────────────────────────────────┤
│  模块层 (modules/) - 命令执行                                │
├─────────────────────────────────────────────────────────────┤
│  解析器层 (resolver/) - 参数解析                             │
├─────────────────────────────────────────────────────────────┤
│  服务层 (services/) - 构建操作                               │
├─────────────────────────────────────────────────────────────┤
│  工具层 (util/) - 辅助函数                                   │
├─────────────────────────────────────────────────────────────┤
│  基础设施层 (containers/, resources/, helper/)              │
└─────────────────────────────────────────────────────────────┘
```

### 根目录核心组件

| 文件 | 用途 | 关键类/函数 |
|------|------|------------|
| `__main__.py` | 入口点引导程序 | `is_in_ohos_dir()`、动态导入 main.py |
| `main.py` | 协调器和 DI 容器 | `Main` 类、模块初始化器、`_is_indep_build()` |
| `setup.py` | Python 包设置 | 包元数据、`hb` 命令的 entry_points |

## 目录结构

### 1. containers/ - 值对象和枚举

**用途：** 构建系统的数据容器、枚举和装饰器。

| 文件 | 职责 |
|------|------|
| `arg.py` | 核心参数系统：`Arg` 类、`ModuleType` 枚举、`BuildPhase` 枚举 |
| `status.py` | 异常处理装饰器 `@throw_exception`、`judge_indep()` 函数 |
| `colors.py` | 终端输出的 ANSI 颜色代码 |

**关键设计模式：**
- **枚举模式：** `ModuleType`、`BuildPhase`、`ArgType`、`CleanPhase` 用于类型安全常量
- **值对象模式：** `Arg` 类封装参数元数据（名称、类型、阶段、解析器）
- **装饰器模式：** `@throw_exception` 用于集中异常处理

### 2. exceptions/ - 异常层次结构

**用途：** 带有错误代码分类的自定义异常类型。

| 文件 | 职责 |
|------|------|
| `ohos_exception.py` | `OHOSException` 类，从 `status.json` 查找错误代码 |

**错误代码分类：**
- 第一位数字表示构建阶段：'1'=预加载器、'2'=加载器、'3'=GN、'4'=ninja
- 代码映射到 `resources/status/status.json` 中的解决方案

### 3. helper/ - 元类和工具

**用途：** 用于强制执行设计约束的元类。

| 文件 | 职责 |
|------|------|
| `singleton.py` | 单例类的 `Singleton` 元类 |
| `no_instance.py` | 纯静态类的 `NoInstance` 元类 |
| `separator.py` | 菜单分组的 `Separator` 类 |

**关键设计模式：**
- **单例模式：** Config 类使用 Singleton 实现全局状态
- **静态类模式：** LogUtil、IoUtil 使用 NoInstance 防止实例化

### 4. modules/ - 命令执行层

**用途：** 为每个 hb 子命令实现命令执行。

#### 接口文件 (modules/interface/)
| 文件 | 职责 |
|------|------|
| `module_interface.py` | 基础 `ModuleInterface` - 所有模块都继承此接口 |
| `build_module_interface.py` | 带构建阶段模板方法模式的 `BuildModuleInterface` |
| `set_module_interface.py` | `hb set` 命令的接口 |
| `clean_module_interface.py` | `hb clean` 命令的接口 |
| `env_module_interface.py` | `hb env` 命令的接口 |
| `tool_module_interface.py` | `hb tool` 命令的接口 |
| `indep_build_module_interface.py` | 独立构建的接口 |
| `install_module_interface.py` | `hb install` 命令的接口 |
| `package_module_interface.py` | `hb package` 命令的接口 |
| `publish_module_interface.py` | `hb publish` 命令的接口 |
| `update_module_interface.py` | `hb update` 命令的接口 |
| `push_module_interface.py` | `hb push` 命令的接口 |

#### 实现文件
| 文件 | 职责 | 关键方法 |
|------|------|----------|
| `ohos_build_module.py` | 标准构建执行 | `_preload()`、`_load()`、`_target_generate()`、`_target_compilation()` |
| `ohos_indep_build_module.py` | 独立组件构建 | `_run_prebuilts()`、`_run_hpm()`、`_run_indep_build()` |
| `ohos_set_module.py` | 产品/配置选择 | `set_product()`、`set_parameter()` |
| `ohos_clean_module.py` | 构建输出清理 | - |
| `ohos_env_module.py` | 环境设置/检查 | - |
| `ohos_tool_module.py` | GN 工具命令 | - |
| `ohos_install_module.py` | HPM 包安装 | - |
| `ohos_package_module.py` | HPM 包创建 | - |
| `ohos_publish_module.py` | HPM 包发布 | - |
| `ohos_update_module.py` | HPM 包更新 | - |
| `ohos_push_module.py` | 通过 HDC 推送到设备 | - |

### 5. resolver/ - 参数解析层

**用途：** 解析 CLI 参数并将其解析为构建配置。

#### 接口
| 文件 | 职责 |
|------|------|
| `interface/args_resolver_interface.py` | `ArgsResolverInterface` 基类，带有 `resolve_arg()` 方法 |

#### 解析器
| 文件 | 职责 | 关键方法 |
|------|------|----------|
| `build_args_resolver.py` | 解析构建参数 | `resolve_product()`、`resolve_ccache()`、`resolve_gn_args()`、`resolve_build_target()` |
| `set_args_resolver.py` | 解析 set 参数 | `resolve_product_name()`、`resolve_all()` |
| `clean_args_resolver.py` | 解析 clean 参数 | - |
| `env_args_resolver.py` | 解析 env 参数 | - |
| `tool_args_resolver.py` | 解析 tool 参数 | - |
| `indep_build_args_resolver.py` | 解析独立构建参数 | `get_part_name()` |
| `install_args_resolver.py` | 解析 install 参数 | - |
| `package_args_resolver.py` | 解析 package 参数 | - |
| `publish_args_resolver.py` | 解析 publish 参数 | - |
| `update_args_resolver.py` | 解析 update 参数 | - |
| `push_args_resolver.py` | 解析 push 参数 | - |
| `judge_indep_args_resolver.py` | 确定构建是否为独立构建 | `is_indep_args()` |

#### 工厂
| 文件 | 职责 |
|------|------|
| `args_factory.py` | 从 JSON 定义创建 argparse 选项 |

### 6. services/ - 核心构建服务

**用途：** 封装构建操作和外部工具调用。

#### 接口文件 (services/interface/)
| 文件 | 职责 |
|------|------|
| `service_interface.py` | 基础 `ServiceInterface`，带有 `args_dict`、`exec` 属性 |
| `preload_interface.py` | 预加载器服务的接口 |
| `load_interface.py` | 加载器服务的接口 |
| `build_file_generator_interface.py` | GN/HPM 构建文件生成的接口 |
| `build_executor_interface.py` | Ninja 构建执行的接口 |
| `menu_interface.py` | 交互式菜单的接口 |
| `prebuilt_sdk_interface.py` | 预构建 SDK 处理的接口 |

#### 服务实现
| 文件 | 职责 | 关键方法 |
|------|------|----------|
| `preloader.py` | 预加载产品配置 | `_generate_platforms_build()`、`_generate_features_json()`、`_generate_parts_json()`、`_generate_build_config_json()` |
| `loader.py` | 加载子系统/部件信息 | `_check_args()`、`_generate_target_gn()`、`_generate_syscap_files()`、`_generate_system_capabilities()` |
| `gn.py` | GN 构建文件生成 | `execute_gn_cmd()`、`_execute_gn_gen_cmd()`、`_execute_gn_path_cmd()`、`_execute_gn_desc_cmd()` |
| `ninja.py` | Ninja 构建执行 | `_execute_ninja_cmd()`、`_regist_ninja_path()` |
| `hpm.py` | HPM（鸿蒙包管理器）集成 | `execute_hpm_cmd()`、`_execute_hpm_build_cmd()`、`_execute_hpm_install_cmd()` |
| `indep_build.py` | 独立构建协调 | `run()`、`_convert_flags()`、`_generate_dependences_json()` |
| `prebuilts.py` | 预构建二进制文件处理 | - |
| `prebuilt_sdk.py` | 预构建 SDK 编译 | `should_build_sdk()`、`run()` |
| `menu.py` | 交互式产品选择 | `select_product()`、`select_compile_option()`、`_list_promt()` |
| `hdc.py` | HDC（华为调试客户端）集成 | - |

### 7. util/ - 工具函数

**用途：** 按功能区域组织的辅助工具。

#### 根目录工具
| 文件 | 职责 |
|------|------|
| `log_util.py` | 日志工具：`LogUtil.hb_info()`、`hb_warning()`、`hb_error()` |
| `system_util.py` | 系统操作：`SystemUtil.exec_command()`、`get_current_time()` |
| `io_util.py` | 文件 I/O：`IoUtil.read_json_file()`、`dump_json_file()` |
| `type_check_util.py` | 类型验证工具 |
| `component_util.py` | 组件/包操作 |
| `product_util.py` | 产品发现和管理 |
| `device_util.py` | 设备操作 |
| `timer_util.py` | 计时装饰器 |
| `monitor.py` | 构建监控 |

#### 加载器工具 (util/loader/)
| 文件 | 职责 |
|------|------|
| `load_ohos_build.py` | 解析 ohos.build 文件 |
| `load_bundle_file.py` | 解析 bundle.json 文件 |
| `subsystem_info.py` | 子系统信息管理 |
| `subsystem_scan.py` | 子系统扫描 |
| `generate_targets_gn.py` | 生成 BUILD.gn 文件 |
| `platforms_loader.py` | 平台配置加载 |
| `merge_platform_build.py` | 平台构建设置合并 |

#### 预加载器工具 (util/preloader/)
| 文件 | 职责 |
|------|------|
| `preloader_process_data.py` | 预加载器的数据处理 |
| `parse_lite_subsystems_config.py` | 解析精简版子系统配置 |
| `parse_vendor_product_config.py` | 解析厂商/产品配置 |

#### 预构建工具 (util/prebuild/)
| 文件 | 职责 |
|------|------|
| `patch_process.py` | 补丁应用 |

#### 构建后工具 (util/post_build/)
| 文件 | 职责 |
|------|------|
| `part_rom_statistics.py` | ROM 大小统计 |

### 8. resources/ - 配置和数据

**用途：** 配置文件、JSON 模式和参数定义。

| 文件/目录 | 职责 |
|----------|------|
| `config.py` | `Config` 单例 - 集中配置管理 |
| `global_var.py` | 全局常量：路径、文件名、VERSION |
| `args/default/*.json` | 每个命令的默认参数定义 |
| `config/config.json` | 默认构建配置模板 |
| `status/status.json` | 错误代码定义和解决方案 |

**参数 JSON 文件：**
- `buildargs.json` - 构建命令参数
- `setargs.json` - Set 命令参数
- `cleanargs.json` - Clean 命令参数
- `envargs.json` - Env 命令参数
- `toolargs.json` - Tool 命令参数
- `indepbuildargs.json` - 独立构建参数
- `installargs.json` - Install 命令参数
- `packageargs.json` - Package 命令参数
- `publishargs.json` - Publish 命令参数
- `updateargs.json` - Update 命令参数
- `pushargs.json` - Push 命令参数

### 9. test/ - 单元测试

**用途：** 服务的单元测试。

| 文件 | 职责 |
|------|------|
| `unitTest/services/preloader_test.py` | 预加载器单元测试 |
| `unitTest/services/loader_test.py` | 加载器单元测试 |
| `unitTest/services/test.py` | 通用测试 |

## 设计模式

### 1. **外观模式** (`__main__.py`)
通过隐藏工作空间验证和动态模块加载的复杂性来简化入口点。

### 2. **工厂方法模式** (`main.py`)
每个命令都有专用的工厂方法：
- `_init_build_module()` - 标准构建，包含预加载器→加载器→GN→Ninja 管道
- `_init_indep_build_module()` - 带 HPM 集成的独立构建
- `_init_tool_module()` - 单工具执行

### 3. **依赖注入**
服务通过构造函数注入而不是内部实例化：
```python
preloader = OHOSPreloader()
loader = OHOSLoader()
generate_ninja = Gn()
ninja = Ninja()
return OHOSBuildModule(args_dict, resolver, preloader, loader, generate_ninja, ninja)
```

### 4. **策略模式** (模块选择)
`module_initializers` 字典将命令映射到工厂方法：
```python
module_initializers = {
    'build': main._init_indep_build_module if main._is_indep_build() else main._init_build_module,
    'set': main._init_set_module,
    'clean': main._init_clean_module,
    # ... 共 11 个命令
}
```

### 5. **模板方法模式** (`BuildModuleInterface`)
定义构建阶段序列：
```python
def run(self):
    self._prebuild_and_preload()
    self._load()
    self._gn()
    self._ninja()
    self._post_target_compilation()
    self._post_build()
```

### 6. **单例模式** (`helper/singleton.py`)
`Config` 类确保单一的全局配置实例。

### 7. **装饰器链**
`@throw_exception` + `@build_tracker` 提供横切关注点（错误处理、指标）。

## 数据和控制流

### 标准构建流程

```
1. CLI 调用
   └─> hb build --product rk3568

2. 入口验证 (__main__.py)
   └─> is_in_ohos_dir() - 向上遍历目录树查找标记文件

3. 协调 (main.py::Main.main())
   ├─> 解析参数 (Arg.parse_all_args)
   ├─> 预构建 SDK 检查 (_prebuild_ohos_sdk)
   ├─> 通过 module_initializers 字典进行模块选择
   └─> 执行 module.run()

4. 构建管道 (modules/)
   ├─> PRE_BUILD: 设置产品、预加载器
   ├─> PRE_LOAD: 预加载配置  
   ├─> LOAD: 加载子系统和部件
   ├─> GN: 生成 ninja 文件
   └─> NINJA: 执行编译

5. 完成
   └─> 记录耗时、清理
```

### 独立构建流程

```
hb build component_name -i

1. 在参数位置检测 -i 标志
2. 验证 indep_configs/build_indep.sh 是否存在
3. 使用 HPM + PrebuiltsService 初始化 IndepBuildModule
4. 委托 HPM 进行组件解析和构建
```

### 参数解析流程

```
1. Arg.parse_all_args(ModuleType) 读取 JSON 定义
2. ArgsFactory 创建 argparse 选项
3. 解析 CLI 参数
4. ArgsResolver 将参数映射到解析函数
5. 每个阶段触发该阶段参数的 resolve_arg()
6. 解析器通过 regist_arg() 配置服务
```

## 集成点

### 上游依赖
- **操作系统**: `os`、`sys`、`subprocess`、`platform`
- **标准库**: `json`、`shutil`、`glob`、`importlib`
- **第三方**: `prompt_toolkit`（交互式菜单）、`kconfiglib`、`PyYAML`、`requests`

### 下游消费者（创建和注入的依赖）

| 层 | 注入的组件 |
|----|-----------|
| **模块** | `OHOSBuildModule`、`OHOSSetModule`、`OHOSCleanModule` 等 |
| **解析器** | `BuildArgsResolver`、`SetArgsResolver`、`CleanArgsResolver` 等 |
| **服务** | `OHOSPreloader`、`OHOSLoader`、`Gn`、`Ninja`、`Hpm`、`Hdc` |

### 调用的外部工具
- **GN**: `prebuilts/build-tools/{platform}-{arch}/bin/gn`
- **Ninja**: `prebuilts/build-tools/{platform}-{arch}/bin/ninja`
- **HPM**: `prebuilts/hpm/node_modules/.bin/hpm` 或系统 PATH
- **HDC**: 设备调试客户端
- **CCache**: 可选的编译器缓存

### 横切关注点
- **containers/**: 值对象（`Arg`、`Colors`）、异常装饰器（`throw_exception`）
- **exceptions/**: 带有错误代码分类的 `OHOSException`
- **resources/**: 配置单例（`Config`）、全局变量、参数 JSON
- **helper/**: 元类（`Singleton`、`NoInstance`）、分隔符

## 模块目录参考

| 目录 | 用途 | 参见 |
|------|------|------|
| `containers/` | 值对象、枚举、装饰器 | [containers/codemap.md](containers/codemap.md) |
| `exceptions/` | 自定义异常层次结构 | [exceptions/codemap.md](exceptions/codemap.md) |
| `helper/` | 元类和工具 | [helper/codemap.md](helper/codemap.md) |
| `modules/` | 命令执行实现 | [modules/codemap.md](modules/codemap.md) |
| `resolver/` | 参数解析和解析 | [resolver/codemap.md](resolver/codemap.md) |
| `resources/` | 配置和 JSON 定义 | [resources/codemap.md](resources/codemap.md) |
| `services/` | 核心构建服务实现 | [services/codemap.md](services/codemap.md) |
| `util/` | 辅助工具和加载器 | [util/codemap.md](util/codemap.md) |

## 关键设计决策

1. **工作空间验证**：入口点在加载前验证 OHOS 源码树结构，以防止运行时错误
2. **动态加载**：`__main__.py` 使用 `importlib` 从项目目录加载 `main.py`，实现上下文切换
3. **独立构建检测**：通过 `judge_indep()` 和 `-i` 标志检测实现双模式支持（源码 vs 组件）
4. **错误处理策略**：通过 `@throw_exception` 装饰器集中处理，错误元数据外置在 JSON 中
5. **配置持久化**：参数存储为 JSON 文件，实现跨阶段状态共享
6. **基于阶段的执行**：构建过程分为多个阶段（PRE_BUILD、PRE_LOAD、LOAD 等）以实现细粒度控制
7. **服务注入**：通过构造函数注入依赖，以提高可测试性和松耦合
8. **交互式菜单支持**：在 `hb set` 中使用 `prompt_toolkit` 进行产品选择 UI

## 文件数量汇总

| 类别 | 数量 | 描述 |
|------|------|------|
| Python 文件 | ~85 | 主要实现 |
| 接口文件 | ~15 | 抽象基类 |
| JSON 配置 | ~15 | 参数定义、状态代码 |
| 测试文件 | ~3 | 单元测试 |

## 入口点

1. **hb 命令**: 在 `setup.py` 中定义为 `hb=hb.__main__:main`
2. **模块执行**: `python -m hb` 触发 `hb/__main__.py`
3. **直接执行**: `python main.py`（需要 OHOS_ROOT 上下文）
