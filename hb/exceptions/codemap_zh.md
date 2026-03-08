# 代码映射：exceptions/ 模块

## 概述

| 属性 | 值 |
|-----------|-------|
| **目录** | `/home/zhuojw/a_git/ohos-mani-v2/build/hb/exceptions/` |
| **主模块** | `ohos_exception.py` |
| **用途** | HarmonyOS 构建系统 (hb) 的集中式异常处理框架 |
| **架构模式** | 分层异常类 + 外部化错误元数据 |

---

## 1. 职责

`exceptions/` 模块为 hb 构建系统定义了**基础错误处理基础设施**：

1. **基础异常定义**：提供 `OHOSException`，所有 hb 特定错误的根异常类
2. **错误代码系统**：实现按编译阶段组织的数字错误代码分类法（4 位代码）
3. **外部化错误元数据**：通过 JSON 配置将错误消息、类型、描述和解决方案与代码解耦
4. **与日志集成**：与 `containers/status.py` 配合，提供格式化的错误报告和诊断输出

### 错误代码分类法

| 代码范围 | 编译阶段 | 描述 |
|------------|-------------------|-------------|
| `0000-0999` | 通用/预构建 | 配置、参数验证、I/O 错误 |
| `1000-1999` | 预加载器 | 预加载器阶段错误 |
| `2000-2999` | 加载器 | 加载器阶段错误（子系统/组件加载） |
| `3000-3999` | GN | GN 元构建生成错误 |
| `4000-4999` | Ninja | Ninja 构建执行错误 |

---

## 2. 设计模式

### 2.1 分层异常架构

```
BaseException (Python 内置)
    └── Exception (Python 内置)
            └── OHOSException (hb 根异常)
                    └── [隐式: 所有 hb 特定错误]
```

**模式**：该模块使用**扁平异常层次结构**，只有一个基类。与其创建许多子类，它使用**错误代码区分**来识别错误类型。

### 2.2 外部化错误元数据（关注点分离）

`OHOSException` 类实现了**数据驱动错误解析**模式：

- 错误元数据（类型、描述、解决方案）存储在外部的 `resources/status/status.json` 中
- 错误实例通过数字 `code` 参数标识
- 运行时查找提供人类可读的错误上下文

**优点**：
- 错误消息可以在不修改代码的情况下更新
- 支持本地化
- 集中式错误文档

### 2.3 基于装饰器的异常转换

伴随模块 `containers/status.py` 提供了 `@throw_exception` 装饰器，实现**异常转换**：

- 捕获 `OHOSException` 和通用 `Exception`
- 将所有异常规范化为一致的错误格式
- 处理退出码传播（`exit(-1)`）
- 提供格式化的堆栈跟踪日志记录

### 2.4 单例错误配置

`STATUS_FILE` 路径定义在 `resources/global_var.py` 中：
```python
STATUS_FILE = os.path.join(CURRENT_HB_DIR, 'resources/status/status.json')
```

---

## 3. 数据与控制流

### 3.1 异常传播流程

```
┌─────────────────────────────────────────────────────────────────────┐
│                         异常传播                        │
└─────────────────────────────────────────────────────────────────────┘

  [检测到错误]
         │
         ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  raise           │────▶│  @throw_exception │────▶│  status.json     │
│  OHOSException(  │     │  装饰器        │     │  查找          │
│    message,      │     │  (containers/     │     │  (get_solution/  │
│    code)         │     │   status.py)      │     │   get_type/      │
└──────────────────┘     └──────────────────┘     │   get_desc)      │
                                                  └──────────────────┘
                                                           │
          ┌──────────────────────────────────────────────────┘
          ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  LogUtil.write_  │────▶│  build.log       │     │  格式化       │
│  log()           │     │  (out/build.log) │────▶│  标准错误输出   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                           │
                                                           ▼
                                                  ┌──────────────────┐
                                                  │  exit(-1)        │
                                                  └──────────────────┘
```

### 3.2 错误代码解析流程

```python
# 错误元数据查找序列：
raise OHOSException("message", "2001")
         │
         ▼
OHOSException.__init__(message, code=2001)
         │
         ▼
get_solution() ──▶ open(STATUS_FILE) ──▶ json.load()
get_type()     ──▶ status_file[str(code)]['type']
get_desc()     ──▶ status_file[str(code)]['description']
```

### 3.3 Status.json 模式

```json
{
  "CODE": {
    "code": "CODE",
    "type": "错误类别/类型",
    "pattern": "自动检测的正则表达式模式",
    "description": "人类可读的描述",
    "solution": "修复步骤（字符串或数组）"
  }
}
```

---

## 4. 集成点

### 4.1 模块依赖（入站）

| 消费者模块 | 导入位置 | 使用模式 |
|-----------------|-----------------|---------------|
| `main.py` | 第 39 行 | 模块验证错误 |
| `containers/arg.py` | 第 48 行 | 参数类型验证 |
| `containers/status.py` | 第 22 行 | 异常处理装饰器 |
| `modules/ohos_*_module.py` (8 个文件) | 各处 | 模块特定错误 |
| `modules/interface/*_interface.py` (6 个文件) | 各处 | 接口错误处理 |
| `resolver/*_args_resolver.py` (8 个文件) | 各处 | 参数解析错误 |
| `resolver/args_factory.py` | 第 21 行 | 参数工厂错误 |
| `services/gn.py` | 第 28 行 | GN 服务错误 |
| `services/ninja.py` | 第 23 行 | Ninja 执行错误 |
| `services/loader.py` | 第 24 行 | 加载器服务错误 |
| `services/hpm.py` | 第 28 行 | HPM (HarmonyOS 包管理器) 错误 |
| `services/menu.py` | 第 36 行 | 交互式菜单错误 |
| `services/hdc.py` | 第 28 行 | HDC (HarmonyOS 设备连接器) 错误 |
| `util/io_util.py` | 第 26 行 | I/O 工具错误 |
| `util/log_util.py` | 第 27 行 | 日志工具错误 |
| `util/device_util.py` | 第 21 行 | 设备工具错误 |
| `util/component_util.py` | 第 26 行 | 组件工具错误 |
| `util/product_util.py` | 第 23 行 | 产品工具错误 |
| `util/loader/*.py` (4 个文件) | 各处 | 加载器工具错误 |
| `util/prebuild/patch_process.py` | 第 21 行 | 补丁处理错误 |
| `util/type_check_util.py` | 第 19 行 | 类型检查错误 |

**总消费者**：35+ 个文件中约 52 个导入点

### 4.2 依赖（出站）

| 依赖 | 用途 |
|------------|---------|
| `json` | 解析 status.json 错误元数据 |
| `resources.global_var.STATUS_FILE` | 错误配置文件路径 |

### 4.3 关键集成模式

#### 模块模式（Try-Except-重新抛出）
```python
# 跨模块的常见模式
from exceptions.ohos_exception import OHOSException

def some_operation():
    try:
        # ... 操作 ...
    except SomeException as e:
        raise OHOSException(f"Context: {e}", "XXXX")
```

#### 解析器模式（验证）
```python
# 参数解析器中的常见模式
if invalid_condition:
    raise OHOSException('ERROR argument "--param": Invalid value', "CODE")
```

#### 装饰器模式（全局处理器）
```python
from containers.status import throw_exception

@throw_exception
def entry_point():
    # 所有异常被捕获、格式化、记录
    raise OHOSException("...", "...")
```

---

## 5. 类参考

### 5.1 OHOSException

**位置**：`exceptions/ohos_exception.py:24`

```python
class OHOSException(Exception):
    """
    HarmonyOS 构建系统错误的根异常类。
    
    属性：
        _code (int): 用于外部元数据查找的数字错误代码
        _message (str): 人类可读的错误消息
    """
    
    def __init__(self, message: str, code: int = 0):
        """
        使用消息和可选错误代码初始化异常。
        
        参数：
            message: 人类可读的错误描述
            code: 数字错误代码（默认：0 = 未知）
        """
    
    def get_solution(self) -> str:
        """
        从 status.json 检索解决方案文本。
        如果找不到代码，返回 'UNKNOWN REASON'。
        """
    
    def get_type(self) -> str:
        """
        从 status.json 检索错误类型/类别。
        如果找不到，返回 'UNKNOWN ERROR TYPE'。
        """
    
    def get_desc(self) -> str:
        """
        从 status.json 检索详细描述。
        如果找不到，返回 'NO DESCRIPTION'。
        """
```

---

## 6. 错误代码注册表（精选）

| 代码 | 类型 | 描述 |
|------|------|-------------|
| `0000` | 未知 | 通用未知错误 |
| `0001` | 预构建 | 缺少预构建依赖项 |
| `0008` | I/O | 文件未找到 (io_util) |
| `0019` | 配置 | 开发环境未初始化 |
| `1001` | 预加载器 | 预加载器结果不正确 |
| `2001-2014` | 加载器 | 各种加载器阶段错误 |
| `3000-3014` | GN | GN 元构建错误（语法、缺少文件、依赖） |
| `4000-4016` | Ninja | 构建执行错误（语法、链接、未定义引用） |

---

## 7. 设计特性

### 优点
1. **集中式错误管理**：错误处理的单一事实来源
2. **可扩展的代码系统**：数字范围允许新的错误类别
3. **解耦的元数据**：无需代码更改即可更新错误解决方案
4. **一致的用户体验**：所有模块的统一错误格式化

### 注意事项
1. **异常时的文件 I/O**：每次元数据查找都会打开/读取/解析 JSON
2. **基于字符串的代码**：错误代码作为字符串传递（"2001" 而不是 2001）
3. **扁平层次结构**：没有用于特定捕获处理的语义异常子类

---

## 8. 相关文件

| 文件 | 关系 |
|------|--------------|
| `containers/status.py` | 异常处理装饰器和错误输出格式化 |
| `resources/global_var.py` | STATUS_FILE 路径常量 |
| `resources/status/status.json` | 错误元数据库 |
| `util/log_util.py` | 错误跟踪的日志写入 |
