# util/ - 工具函数和辅助类

提供设备管理、日志记录、文件操作、系统命令和构建流程阶段等跨领域功能的静态工具类。所有工具类都使用 `NoInstance` 元类来强制纯静态使用。

## 职责

**主要目的：** 为构建系统提供可复用的工具函数，涵盖设备管理、文件操作、日志记录和构建阶段处理。

**核心职责：**
- **设备管理 (`device_util.py`)**：HDC（HarmonyOS Device Connector）集成，用于设备通信
- **日志记录 (`log_util.py`)**：结构化日志，支持文件和控制台输出
- **IO 操作 (`io_util.py`)**：文件读写、JSON 序列化、子进程执行
- **系统工具 (`system_util.py`)**：平台检测、环境变量、路径工具
- **产品工具 (`product_util.py`)**：产品配置解析和验证
- **类型检查 (`type_check_util.py`)**：参数类型验证和转换
- **计时器工具 (`timer_util.py`)**：构建阶段计时和成本追踪
- **构建流程阶段**：
  - `loader/` - 子系统和部件加载
  - `preloader/` - 预构建配置生成
  - `prebuild/` - 预构建二进制文件管理
  - `post_build/` - 打包、签名和分发

## 设计模式

### 1. **静态工具类（NoInstance 模式）**
所有工具类都使用来自 `helper/no_instance.py` 的 `NoInstance` 元类：
```python
class LogUtil(metaclass=NoInstance):
    @staticmethod
    def log_info(message):
        ...
```
这样可以防止意外实例化并明确使用意图。

### 2. **外观模式**
工具类为复杂操作提供简化的接口：
- `LogUtil` 抽象了文件和控制台日志记录
- `IoUtil` 包装文件操作并带有错误处理
- `DeviceUtil` 在 HDC 之上提供高级设备命令

### 3. **模板方法模式**（构建流程）
每个构建阶段子目录遵循一致的模式：
```
preloader/preloader_generate_config.py
preloader/preloader_process_data.py
loader/generate_targets_gn.py
loader/parts_management.py
```

### 4. **计时器装饰器模式** (`timer_util.py`)
```python
@TimerUtil.cost_time
@throw_exception
@build_tracker
def run():
    ...
```
提供跨领域的计时和指标收集功能。

## 数据与控制流

### 工具调用流程

```
1. 服务层 (services/)
   ├─> DeviceUtil.reboot_device() 用于 HDC 命令
   ├─> LogUtil.log_info() 用于构建输出
   ├─> IoUtil.read_json() 用于配置文件
   └─> SystemUtil.get_platform() 用于操作系统检测

2. 模块层 (modules/)
   └─> ProductUtil.get_product_info() 用于产品配置

3. 解析层 (resolver/)
   └─> TypeCheckUtil.validate() 用于参数验证

4. 构建流程
   ├─> Preloader: 生成构建配置
   ├─> Loader: 加载子系统定义
   ├─> Prebuild: 下载依赖
   └─> Post-build: 打包和签名输出
```

### 流程阶段流转

```
预加载阶段 (Preloader Stage)
├─> preloader_generate_config.py: 生成 build_config.json
└─> preloader_process_data.py: 处理产品和部件定义

加载阶段 (Loader Stage)
├─> generate_targets_gn.py: 生成 GN 目标文件
├─> parts_management.py: 管理组件依赖
└─> subsystem_file.py: 加载子系统配置

后构建阶段 (Post-Build Stage)
├─> hap_pack.py: 打包 HAP 文件
├─> generate_signed_bn.py: 生成签名二进制文件
└─> package_dist.py: 创建分发包
```

## 集成点

### 核心工具模块

| 模块 | 关键函数 | 调用方 |
|------|----------|--------|
| `device_util.py` | `reboot_device()`, `install_hap()`, `shell_command()` | services/hdc.py, modules/* |
| `log_util.py` | `log_info()`, `log_error()`, `init_logger()` | 所有层级 |
| `io_util.py` | `read_json()`, `write_file()`, `run_command()` | services/, util/ |
| `system_util.py` | `get_platform()`, `set_env()` | services/, resolver/ |
| `product_util.py` | `get_product_info()`, `get_product_list()` | services/preloader.py |
| `type_check_util.py` | `validate_type()`, `convert_type()` | resolver/ |
| `timer_util.py` | `cost_time` 装饰器 | modules/ |

### 流程子目录

| 目录 | 用途 | 关键文件 |
|------|------|----------|
| `loader/` | 子系统加载 | `generate_targets_gn.py`, `parts_management.py`, `subsystem_file.py` |
| `preloader/` | 预构建配置 | `preloader_generate_config.py`, `preloader_process_data.py` |
| `prebuild/` | 依赖下载 | `prebuilts_download.py` |
| `post_build/` | 打包 | `hap_pack.py`, `generate_signed_bn.py`, `package_dist.py` |

### 上游依赖
- **helper/**: `NoInstance` 元类用于强制静态工具使用
- **resources/**: `Config` 单例用于全局状态，`global_var.py` 用于路径
- **exceptions/**: `OHOSException` 用于错误处理

### 下游调用方
- **services/**: 工具的主要调用方（HDC、预加载器、加载器服务）
- **modules/**: 使用 ProductUtil、TimerUtil
- **resolver/**: 使用 TypeCheckUtil 进行参数验证
- **main.py**: 使用多种工具进行编排

## 关键文件

### 顶层工具
- **`device_util.py`**: HDC 设备通信包装器（约 100 行）
- **`log_util.py`**: 结构化日志，支持文件/控制台双输出
- **`io_util.py`**: 支持 JSON 的文件 I/O 操作
- **`system_util.py`**: 平台检测和环境管理
- **`product_util.py`**: 产品配置解析
- **`component_util.py`**: 组件级操作
- **`type_check_util.py`**: 运行时类型验证
- **`timer_util.py`**: 构建计时和指标
- **`monitor.py`**: 构建监控工具
- **`get_target_info.py`**: 目标平台信息获取

### 流程工具

**loader/**（子系统加载）
- `generate_targets_gn.py`: GN 构建文件生成
- `parts_management.py`: 组件依赖管理
- `parts_config.py`: 部件配置解析
- `platforms_parts.py`: 平台特定部件加载
- `subsystem_file.py`: 子系统定义加载

**preloader/**（预构建配置）
- `preloader_generate_config.py`: 生成 build_config.json
- `preloader_process_data.py`: 处理产品和子系统数据

**prebuild/**（依赖管理）
- `prebuilts_download.py`: 下载和管理预构建二进制文件

**post_build/**（打包和分发）
- `hap_pack.py`: HAP（HarmonyOS Ability Package）创建
- `generate_signed_bn.py`: 二进制签名工具
- `package_dist.py`: 分发包生成
- `package_cc_library.py`: C/C++ 库打包
- `package_entries.py`: 入口点打包

另请参阅：
- [helper/codemap.md](../helper/codemap.md) - NoInstance 元类
- [services/codemap.md](../services/codemap.md) - 主要工具调用方

（文件结束 - 共 175 行）
