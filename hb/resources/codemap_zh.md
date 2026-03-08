# resources/ - 配置与全局状态

集中式配置管理、参数定义以及 HarmonyOS 构建系统的全局状态。为构建范围内的配置持久化实现注册表和单例模式。

## 职责

**主要目的：** 为 hb 构建系统提供配置基础设施和共享状态。

**关键职责：**
- **全局配置 (`config.py`)**：单例模式用于构建配置持久化
- **全局变量 (`global_var.py`)**：模块级常量和路径（状态文件、日志路径、构建目录）
- **参数定义 (`args/`)**：所有 11 个命令的 JSON 驱动 CLI 参数规范
- **状态定义 (`status/`)**：错误代码分类和错误消息注册表
- **构建工具配置 (`build_tools/`)**：外部工具路径和模板

## 设计模式

### 1. **单例模式** (`config.py`)
`Config` 类使用 `helper/singleton.py` 中的 `Singleton` 元类来确保单一全局实例：
```python
class Config(metaclass=Singleton):
    """全局配置注册表。"""
```
被 40 多个文件用于共享构建状态（产品、变体、构建目录等）。

### 2. **注册表模式** (`global_var.py`)
模块级常量充当全局注册表：
- `STATUS_FILE`：错误分类位置
- `DEFAULT_BUILD_LOG`：构建日志路径
- `DEFAULT_BUILD_DIR`：默认输出目录
- `PREBUILTS_DOWNLOAD_SCRIPT`：SDK 下载脚本路径

### 3. **配置即数据** (`args/*.json`)
参数以声明方式在 JSON 中定义，运行时加载：
```json
{
  "args": [
    {
      "arg_name": "product",
      "arg_type": "str",
      "arg_value": "",
      "resolve_function": "resolve_product",
      "help_info": "Build a named product"
    }
  ]
}
```

### 4. **外部化元数据** (`status/status.json`)
错误代码、消息和解决方案存储在外部：
```json
{
  "error_code": "1000",
  "error_type": "Preloader Error",
  "description": "Failed to preload configuration",
  "solution": "Check product configuration"
}
```

## 数据与控制流

### 配置流

```
1. 应用启动
   └─> Config() 实例化（单例）

2. 参数解析（resolver/）
   ├─> 加载 args/*.json 定义
   ├─> 通过 Arg.create_instance_by_dict() 构建 Arg 对象
   └─> 存储在 resolver._args_to_function 注册表中

3. 构建执行
   ├─> Config.set_product(), Config.set_variant()
   ├─> 服务通过 Config.get_*() 查询当前状态
   └─> 通过 JSON 文件实现跨阶段状态持久化

4. 错误处理
   └─> 通过 error_code 从 status.json 查找错误元数据
```

### 配置持久化机制

参数和状态可以作为 JSON 文件持久化到构建输出目录：
```
out/{product}/build_configs/
├── args/           # 序列化的 Arg 对象
├── status/         # 错误状态
└── tools/          # 工具配置
```

## 集成点

### 核心 Python 文件

| 文件 | 模式 | 使用者 |
|------|------|--------|
| `config.py` | 单例 | 40+ 文件（modules/, services/, util/, resolver/） |
| `global_var.py` | 模块常量 | 用于所有层的路径 |

### JSON 配置目录

| 目录 | 内容 | 用途 |
|------|------|------|
| `args/default/` | 11 个命令的参数定义 | CLI 参数规范 |
| `build_tools/` | 外部工具配置 | HPM、Node.js、预构建路径 |
| `status/` | 错误分类 | 错误代码 → 消息/解决方案映射 |

### 依赖注入使用者

- **resolvers/**：从 JSON 加载参数定义
- **services/**：查询 Config 单例获取当前产品/变体
- **modules/**：通过 Config 存储持久状态
- **exceptions/**：从 status.json 查找错误元数据

## 关键文件

### Python 模块
- **`config.py`**：全局配置单例，具有产品、变体、构建目录和自定义参数的 get/set 方法
- **`global_var.py`**：模块级常量，定义状态文件、日志文件、预构建下载脚本和默认目录的路径

### JSON 配置
- **`args/default/buildargs.json`**：构建命令参数定义（产品、变体、任务数等）
- **`args/default/cleanargs.json`**：清理命令参数
- **`args/default/setargs.json`**：设置命令参数
- **`args/default/toolargs.json`**：工具执行参数
- **`status/status.json`**：4 位错误代码分类（0000-4000），按构建阶段组织
- **`build_tools/ohos_build_tools.json`**：HPM 和 Node.js 工具路径

## 错误代码分类

| 代码范围 | 类别 | 描述 |
|----------|------|------|
| 0000-0999 | 通用 | 系统级错误 |
| 1000-1999 | 预加载器 | 产品配置错误 |
| 2000-2999 | 加载器 | 子系统加载错误 |
| 3000-3999 | GN | 构建文件生成错误 |
| 4000-4999 | Ninja | 编译错误 |

另请参阅：
- [containers/codemap.md](../containers/codemap.md) - 参数值对象
- [exceptions/codemap.md](../exceptions/codemap.md) - OHOSException 用法
- [resolver/codemap.md](../resolver/codemap.md) - 参数解析流程
