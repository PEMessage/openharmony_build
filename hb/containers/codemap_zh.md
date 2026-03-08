# 容器模块代码映射

## 1. 职责

`containers/` 模块作为鸿蒙构建系统（hb）的**核心领域模型层**。它定义了基础数据结构、枚举和横切关注点，支撑整个构建系统的类型系统、参数处理和错误处理基础设施。

### 主要功能：
- **类型系统定义**：为构建阶段、模块类型和参数类型定义强类型枚举
- **参数封装**：提供 `Arg` 类作为 CLI 参数的规范表示，并具备生命周期管理能力
- **异常处理**：通过 `throw_exception` 装饰器实现集中式异常处理
- **可视化输出**：提供 ANSI 颜色常量以确保终端格式化的一致性

---

## 2. 设计模式

### 2.1 值对象 / 数据传输对象（DTO）
**文件**：`arg.py` - 类：`Arg`

`Arg` 类实现了**值对象**模式（不可变数据载体），并对解析值提供可控的可变性：

```python
class Arg:
    def __init__(self, name: str, helps: str, phase: str,
                 attribute: dict, argtype: ArgType, value,
                 resolve_function: str)
```

**特性**：
- 封装单个命令行参数的所有元数据
- 基于属性的访问控制（`@property` 装饰器）并带有 `arg_value` 的 setter
- 基于 `arg_name` 的语义相等性（由 `__str__` 返回 `name=value` 暗示）
- 自包含的序列化/反序列化逻辑

### 2.2 静态工厂模式
**文件**：`arg.py` - 方法：`Arg.create_instance_by_dict()`

用于从 JSON 配置构造 `Arg` 实例的工厂方法：
- 解析 `resources/args/default/*.json` 中的 JSON 模式
- 基于 `ArgType` 映射执行类型强制转换
- 处理参数名称的中划线到下滑线的转换
- 通过 `OHOSException` 验证未知类型

### 2.3 枚举模式
**文件**：`arg.py`

三个类枚举提供类型安全的常量：

| 类 | 模式 | 用途 |
|-------|---------|---------|
| `ModuleType(Enum)` | 标准 Python 枚举 | 11 个构建命令类别（BUILD、SET、ENV、CLEAN 等） |
| `ArgType` | 带有静态常量的类 | 参数类型系统（BOOL、INT、STR、LIST、DICT、SUBPARSERS） |
| `BuildPhase` | 带有静态常量的类 | 13 阶段构建生命周期 |
| `CleanPhase` | 带有静态常量的类 | 清理操作模式（REGULAR、DEEP、NONE） |

**类型解析策略**：
每个非枚举类提供 `get_type(value: str)` 静态方法用于字符串到常量的映射，从而支持从 JSON 配置文件反序列化。

### 2.4 装饰器模式
**文件**：`status.py` - 函数：`throw_exception`

为横切错误处理实现**面向切面编程（AOP）**：

```python
def throw_exception(func):
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except OHOSException and Exception as exception:
            # 集中式错误格式化和日志记录
            _print_formatted_tracebak(...)
            exit(-1)
    return wrapper
```

**处理切面**：
- 异常拦截和分类
- 向控制台输出格式化错误
- 结构化日志记录到 `out/build.log`
- 以错误码终止进程

### 2.5 数据类值对象
**文件**：`colors.py` - 类：`Colors`

使用 Python `@dataclass` 装饰器的简单**值对象**：
- 仅包含类级常量（ANSI 转义码）
- 无实例状态；用作颜色常量的命名空间
- 由 `LogUtil` 用于一致的终端样式

---

## 3. 数据与控制流

### 3.1 参数初始化流程

```
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 1：参数定义（静态）                                               │
│  resources/args/default/{module}args.json                             │
│  → 定义 arg_name、arg_type、argDefault 等的 JSON 模式                  │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 2：反序列化                                                     │
│  Arg.read_args_file(module_type)                                     │
│  → 检查 CURRENT_ARGS_DIR 是否存在                                     │
│  → 如果当前参数不存在则复制默认 JSON                                   │
│  → 通过 IoUtil.read_json_file() 返回解析后的字典                       │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 3：对象实例化                                                   │
│  Arg.create_instance_by_dict(json_entry)                             │
│  → 通过 ArgType.get_type() 进行类型强制转换                           │
│  → 通过 BuildPhase.get_type() 进行阶段映射                            │
│  → 基于 arg_type 的默认值类型转换                                     │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 4：CLI 解析                                                     │
│  Arg.parse_all_args(module_type)                                     │
│  → ArgsFactory.genetic_add_option() 构建 argparse                    │
│  → parser.parse_known_args() 提取用户值                               │
│  → TypeCheckUtil.tile_list() 扁平化 LIST/SUBPARSERS                   │
│  → Arg.write_args_file() 持久化到 CURRENT_ARGS_DIR                    │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 异常处理流程

```
┌─────────────────────────────────────────────────────────────────────┐
│  被装饰函数执行                                                       │
│  @throw_exception                                                    │
│  def risky_operation():                                              │
│      raise OHOSException("Error", "0001")                            │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼ （抛出异常）
┌─────────────────────────────────────────────────────────────────────┐
│  throw_exception.wrapper()                                           │
│  → 捕获 OHOSException 或通用 Exception                                │
│  → judge_indep() 检查是否为独立构建模式                                │
│  → 如果是 indep：原始异常 + traceback.print_exc()                     │
│  → 否则：通过 _print_formatted_tracebak() 格式化错误                   │
└─────────────────────────────────────────────────────────────────────┘
                               │
                               ▼
┌─────────────────────────────────────────────────────────────────────┐
│  _print_formatted_tracebak()                                         │
│  → 从 ROOT_CONFIG_FILE 或默认值解析日志路径                             │
│  → 将完整堆栈跟踪写入 build.log                                       │
│  → 写入结构化错误报告（代码、原因、类型等）                              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 颜色使用流程

```
LogUtil.hb_warning() / LogUtil.write_log()
        │
        ▼
Colors.WARNING / Colors.ERROR / Colors.INFO
        │
        ▼
ANSI 转义码前置到输出字符串
        │
        ▼
带颜色格式的终端渲染
```

---

## 4. 集成点

### 4.1 上游依赖（containers 使用）

| 依赖 | 使用位置 | 用途 |
|------------|----------------|---------|
| `resources.global_var` | `arg.py` | 参数 JSON 文件的文件路径常量 |
| `exceptions.ohos_exception.OHOSException` | `arg.py`, `status.py` | 领域特定异常类型 |
| `util.log_util.LogUtil` | `arg.py`, `status.py` | 结构化日志记录到 build.log |
| `util.io_util.IoUtil` | `arg.py`, `status.py` | JSON 文件 I/O 操作 |
| `util.type_check_util.TypeCheckUtil` | `arg.py` | 列表扁平化和类型验证 |
| `resolver.args_factory.ArgsFactory` | `arg.py` | argparse 选项生成 |

### 4.2 下游使用者（containers 的用户）

#### 核心模块使用者：

| 使用者文件 | 导入的符号 | 使用上下文 |
|---------------|------------------|---------------|
| `main.py` | `Arg`, `ModuleType`, `throw_exception`, `judge_indep` | 入口点参数路由 |
| `modules/ohos_build_module.py` | `BuildPhase`, `throw_exception` | 构建生命周期编排 |
| `modules/ohos_clean_module.py` | `CleanPhase` | 清理操作模式选择 |
| `modules/ohos_indep_build_module.py` | `BuildPhase` | 独立构建阶段管理 |
| `modules/interface/tool_module_interface.py` | `ModuleType`, `Arg` | 工具模块契约 |

#### 解析器使用者（参数解析）：

| 使用者文件 | 导入的符号 | 用途 |
|---------------|------------------|---------|
| `resolver/build_args_resolver.py` | `Arg`, `throw_exception` | 构建参数解析 |
| `resolver/clean_args_resolver.py` | `Arg` | 清理参数解析 |
| `resolver/env_args_resolver.py` | `Arg`, `ModuleType` | 环境参数解析 |
| `resolver/set_args_resolver.py` | `Arg`, `ModuleType` | 设置参数解析 |
| `resolver/indep_build_args_resolver.py` | `Arg`, `ModuleType` | 独立构建解析 |
| `resolver/install_args_resolver.py` | `Arg`, `ModuleType` | 安装命令解析 |
| `resolver/package_args_resolver.py` | `Arg`, `ModuleType` | 打包命令解析 |
| `resolver/publish_args_resolver.py` | `Arg`, `ModuleType` | 发布命令解析 |
| `resolver/push_args_resolver.py` | `Arg`, `ModuleType` | 推送命令解析 |
| `resolver/update_args_resolver.py` | `Arg`, `ModuleType` | 更新命令解析 |
| `resolver/tool_args_resolver.py` | `Arg`, `throw_exception` | 工具命令解析 |
| `resolver/interface/args_resolver_interface.py` | `Arg`, `throw_exception` | 解析器基础接口 |

#### 服务层使用者：

| 使用者文件 | 导入的符号 | 用途 |
|---------------|------------------|---------|
| `services/gn.py` | `Arg`, `ModuleType`, `throw_exception` | GN 构建系统集成 |
| `services/loader.py` | `throw_exception` | 构建配置加载 |
| `services/menu.py` | `Arg`, `ModuleType` | 交互式菜单系统 |
| `services/hpm.py` | `throw_exception` | HPM 包管理器集成 |
| `services/hdc.py` | `throw_exception`, `Arg`, `ModuleType` | HDC 设备通信 |
| `services/prebuilt_sdk.py` | `Arg` | 预构建 SDK 管理 |

#### 工具使用者：

| 使用者文件 | 导入的符号 | 用途 |
|---------------|------------------|---------|
| `util/log_util.py` | `Colors` | 终端颜色格式化 |
| `util/product_util.py` | `throw_exception` | 产品配置工具 |
| `util/system_util.py` | `throw_exception` | 系统级操作 |
| `util/device_util.py` | `throw_exception` | 设备管理工具 |
| `util/component_util.py` | `throw_exception` | 组件工具 |
| `util/loader/*.py` | `throw_exception` | 各种加载器工具 |

### 4.3 文件系统集成

#### 配置文件布局：

```
resources/args/default/
├── buildargs.json      # 默认构建参数 (ModuleType.BUILD)
├── setargs.json        # 默认设置参数 (ModuleType.SET)
├── cleanargs.json      # 默认清理参数 (ModuleType.CLEAN)
├── envargs.json        # 默认环境参数 (ModuleType.ENV)
├── toolargs.json       # 默认工具参数 (ModuleType.TOOL)
├── indepbuildargs.json # 默认独立构建参数 (ModuleType.INDEP_BUILD)
├── installargs.json    # 默认安装参数 (ModuleType.INSTALL)
├── packageargs.json    # 默认打包参数 (ModuleType.PACKAGE)
├── publishargs.json    # 默认发布参数 (ModuleType.PUBLISH)
├── updateargs.json     # 默认更新参数 (ModuleType.UPDATE)
└── pushargs.json       # 默认推送参数 (ModuleType.PUSH)

out/hb_args/            # 运行时参数状态目录
├── buildargs.json      # 当前构建参数（从默认复制）
├── setargs.json        # 当前设置参数
└── ...                 # 其他模块当前参数
```

#### 状态持久化模型：
- **默认值**：`resources/args/default/` 中的不可变配置
- **当前状态**：`out/hb_args/` 中的可变运行时状态
- **生命周期**：参数从默认复制 → 解析 → 修改 → 持久化返回

---

## 5. 模块类型矩阵

| ModuleType | 默认参数文件 | 当前参数文件 | 主解析器 | 模块类 |
|------------|-------------------|-------------------|------------------|--------------|
| BUILD | buildargs.json | buildargs.json | build_args_resolver.py | ohos_build_module.py |
| SET | setargs.json | setargs.json | set_args_resolver.py | ohos_set_module.py |
| ENV | envargs.json | envargs.json | env_args_resolver.py | ohos_env_module.py |
| CLEAN | cleanargs.json | cleanargs.json | clean_args_resolver.py | ohos_clean_module.py |
| TOOL | toolargs.json | toolargs.json | tool_args_resolver.py | ohos_tool_module.py |
| INDEP_BUILD | indepbuildargs.json | indepbuildargs.json | indep_build_args_resolver.py | ohos_indep_build_module.py |
| INSTALL | installargs.json | installargs.json | install_args_resolver.py | ohos_install_module.py |
| PACKAGE | packageargs.json | packageargs.json | package_args_resolver.py | ohos_package_module.py |
| PUBLISH | publishargs.json | publishargs.json | publish_args_resolver.py | ohos_publish_module.py |
| UPDATE | updateargs.json | updateargs.json | update_args_resolver.py | ohos_update_module.py |
| PUSH | pushargs.json | pushargs.json | push_args_resolver.py | ohos_push_module.py |

---

## 6. 构建阶段生命周期

`BuildPhase` 类定义了一个 13 阶段的构建流水线：

```
PRE_BUILD (1)
    ↓
PRE_LOAD (2) → HPM_DOWNLOAD (11) [替代路径]
    ↓
LOAD (3)
    ↓
PRE_TARGET_GENERATE (4)
    ↓
TARGET_GENERATE (5)
    ↓
POST_TARGET_GENERATE (6)
    ↓
PRE_TARGET_COMPILATION (7)
    ↓
TARGET_COMPILATION (8)
    ↓
POST_TARGET_COMPILATION (9)
    ↓
POST_BUILD (10)
    ↓
INDEP_COMPILATION (12) [仅独立构建]
```

每个 `Arg` 实例可以通过 `arg_phase` 属性与一个或多个阶段关联，实现阶段特定的参数解析。

---

## 7. 错误码约定

`throw_exception` 装饰器使用的错误码中，**第一位数字表示构建阶段**：

| 第一位数字 | 阶段 | 描述 |
|-------------|-------|-------------|
| 0 | 通用 | 初始化、参数解析、通用错误 |
| 1 | 预加载器 | 预构建配置加载 |
| 2 | 加载器 | 构建目标加载 |
| 3 | GN | GN 元构建系统 |
| 4 | Ninja | Ninja 构建执行 |

示例：`OHOSException("...", "0003")` → 参数解析期间的类型错误
示例：`OHOSException("...", "1001")` → 预加载器阶段的错误

---

## 8. 总结

`containers/` 模块是 hb 构建系统的**架构基础**：

1. **类型安全**：强类型枚举防止无效状态转换
2. **配置即数据**：JSON 驱动的参数定义支持声明式 CLI 设计
3. **集中式错误处理**：AOP 风格的异常装饰确保一致的错误用户体验
4. **关注点分离**：纯数据容器（`Arg`、`Colors`）与行为（`throw_exception`）分离

该模块展示了**领域驱动设计（DDD）**原则，在参数管理、状态跟踪和可视化呈现方面具有清晰的边界上下文。
