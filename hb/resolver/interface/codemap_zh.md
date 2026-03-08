# 代码映射：resolver/interface/

## 1. 职责

`resolver/interface/` 目录为 HarmonyOS 构建系统（hb）中的参数解析定义了**抽象契约**。它为不同的构建命令（build、set、clean、env、tool、package、publish、install、update、push、indep_build）建立了标准化的接口，以实现多态参数处理。

**核心目的：**
- 定义 `ArgsResolverInterface` 抽象基类（ABC），所有具体解析器必须实现该基类
- 建立一种**动态分发机制**，将参数名称映射到相应的解析函数
- 为参数解析提供**模板方法**，具有一致的错误处理机制
- 将参数解析与参数解析逻辑解耦

---

## 2. 设计模式

### 2.1 抽象基类（ABC）模式

```python
class ArgsResolverInterface(metaclass=ABCMeta):
```

该接口使用 Python 的 `abc.ABCMeta` 元类来强制：
- **继承契约**：所有具体解析器实现必须继承自 `ArgsResolverInterface`
- **方法签名强制**：子类必须实现参数配置中引用的解析方法

### 2.2 模板方法模式

该接口实现了模板方法结构：

| 方法 | 角色 | 描述 |
|------|------|------|
| `__init__(args_dict)` | 模板 | 通过将参数映射到函数来初始化解析器 |
| `_map_args_to_function()` | 原始操作 | 验证并将参数名称绑定到可调用的解析方法 |
| `resolve_arg()` | 模板入口 | 用于解析特定参数的公共 API |

**执行流程：**
```
ConcreteResolver.__init__(args_dict)
  └── super().__init__(args_dict) [ArgsResolverInterface.__init__]
        └── _map_args_to_function(args_dict)
              ├── 验证每个 Arg 是否有对应的方法
              ├── 绑定 Arg.resolve_function → 实例方法
              └── 存储在 self._args_to_function[arg_name] = method
```

### 2.3 注册表模式

`_args_to_function` 字典充当**方法注册表**：
```python
self._args_to_function = dict()  # 键：arg_name，值：绑定方法
```

这使得在运行时可以以 O(1) 时间复杂度查找解析函数。

### 2.4 策略模式（通过函数绑定）

每个 `Arg` 通过 `resolve_function` 字符串指定其解析策略：
```python
# JSON 中的 Arg 配置
{
    "arg_name": "--product-name",
    "resolve_function": "resolve_product_name"
}
```

该接口将这些字符串绑定到实际方法，允许**声明式策略选择**。

---

## 3. 数据与控制流

### 3.1 数据结构

#### Arg 容器（`containers.arg.Arg`）
| 属性 | 类型 | 用途 |
|------|------|------|
| `arg_name` | str | 规范化参数名称（例如 `product_name`） |
| `arg_value` | Any | 解析后的 CLI 值 |
| `arg_phase` | BuildPhase/[] | 解析参数的构建阶段 |
| `arg_type` | ArgType | BOOL、INT、STR、LIST、DICT、SUBPARSERS |
| `resolve_function` | str | 要调用的解析方法名称 |
| `arg_attribute` | dict | 附加元数据（已弃用、可选、缩写） |

#### 内部状态
```python
self._args_to_function: dict  # {arg_name: bound_method}
```

### 3.2 解析流程

```
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 1：初始化（构造函数）                                          │
├─────────────────────────────────────────────────────────────────────┤
│  ArgsResolverInterface.__init__(args_dict: dict)                    │
│    ├── 创建空的 _args_to_function 注册表                            │
│    └── 调用 _map_args_to_function(args_dict)                        │
│         ├── 遍历 args_dict.values()                                 │
│         ├── 跳过 'sshkey' 参数（安全例外）                          │
│         ├── 验证 hasattr(self, function_name)                       │
│         ├── 验证 callable(getattr(self, function_name))             │
│         ├── 绑定：entity.resolve_function = bound_method            │
│         └── 注册：_args_to_function[args_name] = bound_method       │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  阶段 2：运行时解析                                                  │
├─────────────────────────────────────────────────────────────────────┤
│  resolve_arg(target_arg: Arg, module)                               │
│    ├── 检查：target_arg.arg_name 是否在 _args_to_function 中        │
│    │   └── 如果未找到则抛出 OHOSException(0000)                     │
│    ├── 检查：_args_to_function[arg_name] 是否可调用                 │
│    │   └── 如果不可调用则抛出 OHOSException                         │
│    ├── 获取：resolve_function = _args_to_function[arg_name]         │
│    └── 调用：return resolve_function(target_arg, module)            │
│         └── 执行具体解析器的静态方法                                │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 错误处理流程

`@throw_exception` 装饰器（来自 `containers.status`）包装公共方法和私有方法：

```python
@throw_exception
def resolve_arg(self, target_arg: Arg, module): ...

@throw_exception
def _map_args_to_function(self, args_dict: dict): ...
```

**错误代码：**
| 代码 | 上下文 | 描述 |
|------|--------|------|
| 0000 | `resolve_arg` | 参数未定义解析函数 |
| 0004 | `_map_args_to_function` | 参数不存在解析函数 |

---

## 4. 集成点

### 4.1 上游依赖

| 组件 | 关系 | 用途 |
|------|------|------|
| `containers.arg.Arg` | 数据容器 | 参数定义和值存储 |
| `containers.status.throw_exception` | 装饰器 | 异常处理和日志记录 |
| `exceptions.ohos_exception.OHOSException` | 错误类型 | 结构化错误报告 |

### 4.2 下游实现

**12 个具体解析器类**继承自 `ArgsResolverInterface`：

| 实现 | 模块类型 | 关键解析方法 |
|------|----------|--------------|
| `BuildArgsResolver` | BUILD | `resolve_product`、`resolve_target`、`clean_output_if_needed` |
| `SetArgsResolver` | SET | `resolve_product_name`、`resolve_set_parameter` |
| `CleanArgsResolver` | CLEAN | `resolve_clean`、`clean_output` |
| `EnvArgsResolver` | ENV | `resolve_env`、`resolve_log_level` |
| `ToolArgsResolver` | TOOL | `resolve_tool` |
| `IndepBuildArgsResolver` | INDEP_BUILD | `resolve_target`、`resolve_product` |
| `PackageArgsResolver` | PACKAGE | `resolve_package` |
| `PublishArgsResolver` | PUBLISH | `resolve_publish` |
| `InstallArgsResolver` | INSTALL | `resolve_install` |
| `UpdateArgsResolver` | UPDATE | `resolve_update` |
| `PushArgsResolver` | PUSH | `resolve_push` |
| `ArgsResolver` (judge) | BUILD | `resolve_target` |

### 4.3 集成架构

```
                    ┌─────────────────────────────────────┐
                    │    resolver/interface/              │
                    │  ┌─────────────────────────────┐    │
                    │  │ ArgsResolverInterface (ABC) │    │
                    │  │  - _args_to_function: dict  │    │
                    │  │  - resolve_arg()            │    │
                    │  │  - _map_args_to_function()  │    │
                    │  └─────────────────────────────┘    │
                    └─────────────────┬───────────────────┘
                                      │ 继承
         ┌─────────────────────────────┼─────────────────────────────┐
         │                             │                             │
         ▼                             ▼                             ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ BuildArgsResolver │    │ SetArgsResolver   │    │ CleanArgsResolver │
│ ───────────────── │    │ ───────────────── │    │ ───────────────── │
│ resolve_product() │    │ resolve_product_  │    │ resolve_clean()   │
│ resolve_target()  │    │   name()          │    │ ...               │
│ ...               │    │ resolve_set_      │    │                   │
│                   │    │   parameter()     │    │                   │
└───────────────────┘    └───────────────────┘    └───────────────────┘
         │                             │                             │
         └─────────────────────────────┼─────────────────────────────┘
                                       │ 实例化
                                       ▼
                     ┌─────────────────────────────────────┐
                     │       modules/ (构建系统)            │
                     │  - BuildModuleInterface             │
                     │  - SetModuleInterface               │
                     │  - CleanModuleInterface             │
                     │  - ...                              │
                     └─────────────────────────────────────┘
```

### 4.4 解析方法契约

所有解析方法必须遵循以下签名：
```python
@staticmethod
def resolve_<arg_name>(target_arg: Arg, module: ModuleInterface) -> Any:
    """
    解析参数值并对模块应用副作用。

    参数：
        target_arg: 包含名称、值和元数据的 Arg 实例
        module: 用于访问构建服务的构建模块接口

    返回：
        解析结果（通常为 None；对模块有副作用）
    """
```

### 4.5 配置驱动绑定

参数到方法的绑定通过 JSON 配置文件以**声明式**方式进行：

```json
{
    "product_name": {
        "arg_name": "--product-name",
        "resolve_function": "resolve_product_name",
        "arg_phase": "prebuild",
        "arg_type": "str"
    }
}
```

`Arg.create_instance_by_dict()` 工厂方法解析这些配置，接口验证引用的 `resolve_function` 方法是否存在于具体解析器类上。

---

## 5. 关键设计决策

| 决策 | 理由 |
|------|------|
| **解析器使用静态方法** | 解析逻辑是无状态的；避免实例耦合 |
| **SSH 密钥排除** | 安全：`sshkey` 参数被显式排除在解析映射之外 |
| **运行时方法验证** | 快速失败：在初始化时检测缺失的解析函数，而非运行时 |
| **ModuleInterface 参数** | 依赖注入：允许解析器与构建系统服务交互 |
| **字符串到方法绑定** | 声明式配置允许在不修改代码的情况下添加参数 |

---

## 6. 文件引用

| 文件 | 行数 | 用途 |
|------|------|------|
| `args_resolver_interface.py` | 56 | 定义解析器契约的抽象基类 |

---

*为 HarmonyOS hb 构建系统生成 - 参数解析接口层*

（文件结束 - 共 265 行）
