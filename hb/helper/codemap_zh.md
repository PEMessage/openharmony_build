# helper/ 模块代码地图

## 1. 职责

`helper/` 目录提供基于**元类的工具**，用于控制类实例化行为和 **CLI 格式化**的展示辅助工具。该模块实现了基础的 Python 元编程模式，用于在 HarmonyOS 构建系统 (hb) 中强制执行架构约束。

### 核心功能：

| 文件 | 用途 |
|------|------|
| `singleton.py` | 强制执行**单例约束** - 确保整个应用生命周期中一个类只存在一个对象实例 |
| `no_instance.py` | 强制执行**纯静态约束** - 阻止类实例化，强制使用纯静态方法 |
| `separator.py` | 为 CLI 菜单分隔符和章节分隔符提供**视觉格式化工具** |

---

## 2. 设计模式

### 2.1 元类模式（Python 特有）

这三个工具都利用了 Python 的**元类机制** - 一种面向类的元编程形式，其中元类控制类创建和实例构造。

```
┌─────────────────────────────────────────────────────────────┐
│                    元类层次结构                              │
├─────────────────────────────────────────────────────────────┤
│  type (内置)                                                │
│    ├── Singleton (自定义)  → 控制单例                        │
│    ├── NoInstance (自定义) → 阻止所有实例化                  │
│    └── ABCMeta (标准)      → 抽象基类                        │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 单例模式

**实现**：`Singleton` 元类重写了 `__call__`

```python
class Singleton(type):
    _instances = {}  # 类级注册表

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

**关键特性**：
- **延迟初始化**：实例在首次访问时创建
- **线程安全**（在 CPython 中由于 GIL，虽然没有显式同步）
- **基于注册表**：使用字典映射类 → 实例
- **继承安全**：层次结构中的每个类都有自己的单例

**使用模式**：
```python
class Config(metaclass=Singleton):
    def __init__(self):
        # ... 初始化

# 两个引用指向同一个对象
config1 = Config()
config2 = Config()
assert config1 is config2  # True
```

### 2.3 静态工具类模式（NoInstance 元类）

**实现**：`NoInstance` 元类重写 `__call__` 以抛出 `TypeError`

```python
class NoInstance(type):
    def __call__(self, *args, **kwds):
        raise TypeError('This class can not be instantiation')
```

**架构目的**：
- 强制执行**纯工具类**模式
- 防止无状态辅助类的意外实例化
- 强制开发者使用 `@staticmethod` 装饰器
- 静态设计的运行时强制执行

### 2.4 工厂/建造者模式（Separator）

**实现**：简单的可配置字符串包装器

```python
class Separator(object):
    line = '-' * 15          # 默认短分隔符
    long_line = '-' * 100    # 长分隔符的类常量

    def __init__(self, line=None):
        if line:
            self.line = f'\n{line}'  # 带换行前缀的自定义标签
```

**模式变体**：
- **默认分隔符**：`Separator()` → 产生 `'-' * 15`
- **带标签的分隔符**：`Separator("CompanyName")` → 产生 `'\nCompanyName'`
- **长分隔符**：通过 `Separator.long_line` 访问

---

## 3. 数据与控制流

### 3.1 单例流：配置状态管理

```
┌─────────────────────────────────────────────────────────────────┐
│                     单例生命周期                                 │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  首次调用：Config()                                             │
│       │                                                         │
│       ▼                                                         │
│  Singleton.__call__(Config)                                     │
│       │                                                         │
│       ├── _instances = {} (空)                                  │
│       │                                                         │
│       ▼                                                         │
│  super().__call__() ──► Config.__init__() ──► 实例存储          │
│                                                         │       │
│  后续调用：Config()                                             │
│       │                                                         │
│       ▼                                                         │
│  Singleton.__call__(Config)                                     │
│       │                                                         │
│       └──► 返回缓存实例 ◄───────────────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 NoInstance 流：静态方法强制

```
┌─────────────────────────────────────────────────────────────────┐
│                 NoInstance 强制流程                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  尝试：LogUtil()                                                │
│       │                                                         │
│       ▼                                                         │
│  NoInstance.__call__(LogUtil)                                   │
│       │                                                         │
│       └──► 抛出 TypeError("This class can not be instantiation") │
│                                                                 │
│  正确使用：LogUtil.hb_info("message")                           │
│       │                                                         │
│       ▼                                                         │
│  直接静态方法调用（绕过 __call__）                               │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Separator 流：CLI 菜单构建

```
┌─────────────────────────────────────────────────────────────────┐
│                   Separator 使用流程                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  产品选择菜单 (services/menu.py)                                │
│       │                                                         │
│       ├── 按公司遍历产品                                        │
│       │       │                                                 │
│       │       └── 检测到新公司                                  │
│       │               │                                         │
│       │               ▼                                         │
│       │       Separator(company_name)  ──► "\nHuawei"          │
│       │               │                                         │
│       │               └── 作为视觉分组标题插入                   │
│       │                                                         │
│       └── 列表渲染检查：isinstance(choice, Separator)          │
│               │                                                 │
│               ├── True: 渲染为不可选择的标题                   │
│               └── False: 渲染为可选择的选项                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 控制流总结

| 辅助工具 | 控制点 | 返回值 | 消费模式 |
|----------|--------|--------|----------|
| `Singleton` | `__call__` | 缓存实例 | 状态管理 |
| `NoInstance` | `__call__` | 抛出异常 | 静态工具访问 |
| `Separator` | `__str__` | 格式化字符串 | CLI 渲染 |

---

## 4. 集成点

### 4.1 依赖关系图

```
┌─────────────────────────────────────────────────────────────────┐
│                    集成点                                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  helper/singleton.py                                            │
│       │                                                         │
│       └──► resources/config.py                                  │
│               └──► Config (全局配置单例)                        │
│                                                                 │
│  helper/no_instance.py                                          │
│       │                                                         │
│       ├──► util/log_util.py    ──► LogUtil                     │
│       ├──► util/io_util.py     ──► IoUtil                      │
│       ├──► util/system_util.py ──► SystemUtil, HandleKwargs    │
│       ├──► util/type_check_util.py ──► TypeCheckUtil           │
│       └──► util/product_util.py ──► ProductUtil                │
│                                                                 │
│  helper/separator.py                                            │
│       │                                                         │
│       ├──► main.py             ──► (已导入，未显示用法)        │
│       └──► services/menu.py    ──► 产品选择 UI                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 消费者详情

#### 单例消费者

| 消费者 | 文件 | 用途 | 导入语句 |
|--------|------|------|----------|
| `Config` | `resources/config.py:39` | 全局构建配置注册表 | `from helper.singleton import Singleton` |

**配置单例状态**：
- 构建路径 (`_root_path`, `_out_path`)
- 目标配置 (`_board`, `_kernel`, `_product`)
- 特性标志 (`_os_level`, `_target_os`, `_target_cpu`)

#### NoInstance 消费者

| 消费者 | 文件 | 用途 | 导入语句 |
|--------|------|------|----------|
| `LogUtil` | `util/log_util.py:37` | 日志外观 | `from hb.helper.no_instance import NoInstance` |
| `HandleKwargs` | `util/system_util.py:33` | 关键字参数处理 | `from hb.helper.no_instance import NoInstance` |
| `SystemUtil` | `util/system_util.py:97` | 系统命令执行 | `from hb.helper.no_instance import NoInstance` |
| `IoUtil` | `util/io_util.py:29` | 文件 I/O 操作 | `from hb.helper.no_instance import NoInstance` |
| `TypeCheckUtil` | `util/type_check_util.py:23` | 类型验证 | `from hb.helper.no_instance import NoInstance` |
| `ProductUtil` | `util/product_util.py:30` | 产品元数据查询 | `from hb.helper.no_instance import NoInstance` |

**通用模式**：
```python
class XxxUtil(metaclass=NoInstance):
    @staticmethod
    def operation():
        # 纯函数实现
```

#### Separator 消费者

| 消费者 | 文件 | 用途 | 导入语句 |
|--------|------|------|----------|
| `Menu` | `services/menu.py:38` | 交互式产品选择 UI | `from helper.separator import Separator` |
| `Main` | `main.py:89` | CLI 初始化 | `from helper.separator import Separator` |

**菜单集成**：
```python
# services/menu.py:100
product_key = Separator(company_separator)
# 创建以公司名称为标题的视觉分隔符
```

### 4.3 导入路径变体

代码库中使用了两种导入约定：

| 约定 | 示例 | 上下文 |
|------------|---------|--------|
| 从 hb 相对导入 | `from hb.helper.no_instance import NoInstance` | 用于 `util/` 子目录 |
| 从 helper 绝对导入 | `from helper.singleton import Singleton` | 用于 `resources/` 子目录 |
| 从 helper 绝对导入 | `from helper.separator import Separator` | 用于 `main.py`、`services/` |

### 4.4 强制执行的架构约束

| 元类 | 约束 | 违规后果 |
|------|------|----------|
| `Singleton` | 每个类一个实例 | 多次调用返回相同对象（不可能违规） |
| `NoInstance` | 不允许实例化 | 如果尝试实例化则在运行时抛出 `TypeError` |

---

## 总结

`helper/` 模块是一个**基础架构层**，提供 Python 元编程工具：

1. **`Singleton`**：支持配置的全局状态管理
2. **`NoInstance`**：在工具模块中强制执行静态工具类模式
3. **`Separator`**：通过视觉分组元素促进 CLI 用户体验

这些辅助工具展示了 Python 元类系统的复杂使用，以在运行时强制执行架构约束，确保 HarmonyOS 构建系统中设计模式的一致性。

(文件结束 - 共 293 行)
