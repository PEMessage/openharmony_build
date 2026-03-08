# util/loader/ - 子系统和目标加载

构建设置加载工具，用于解析产品定义、子系统配置，并生成 GN 构建文件。实现构建流程的加载器阶段。

## 职责

**主要目的：** 解析 OHOS 构建配置文件并生成 GN (Generate Ninja) 构建文件。

**关键职责：**
- **目标生成 (`generate_targets_gn.py`)**：生成定义 parts、inner_kits、system_kits 的 .gni 文件
- **Bundle 加载 (`load_bundle_file.py`)**：解析 bundle.json 文件获取组件元数据
- **构建配置 (`load_ohos_build.py`)**：解析定义子系统和部件的 ohos.build 文件
- **平台加载 (`platforms_loader.py`)**：加载平台特定的构建配置
- **子系统扫描 (`subsystem_scan.py`)**：扫描源代码树查找子系统定义
- **子系统信息 (`subsystem_info.py`)**：查询和管理子系统元数据
- **平台合并 (`merge_platform_build.py`)**：合并平台特定的构建配置

## 设计模式

### 1. **模板方法模式**
`generate_targets_gn.py` 使用 Jinja2 模板进行代码生成：
```python
PARTS_LIST_GNI_TEMPLATE = """
parts_list = [
  {}
]
"""
```

### 2. **解析器模式**
针对不同配置格式的多个解析器：
- `load_ohos_build.py`：解析 BUILD.gn 和 ohos.build 文件
- `load_bundle_file.py`：解析 bundle.json (HPM 包元数据)
- `platforms_loader.py`：解析平台配置 JSON 文件

### 3. **注册表模式**
`subsystem_info.py` 维护已加载子系统及其元数据的注册表。

## 数据与控制流

### 加载器流程

```
1. 服务层调用加载器
   └─> services/loader.py 调用加载器工具

2. 配置解析
   ├─> subsystem_scan.py: 扫描子系统目录
   ├─> load_ohos_build.py: 解析 ohos.build 文件
   ├─> load_bundle_file.py: 解析组件的 bundle.json
   └─> platforms_loader.py: 加载平台配置

3. 数据处理
   ├─> subsystem_info.py: 构建子系统元数据注册表
   ├─> merge_platform_build.py: 合并平台特定配置
   └─> generate_targets_gn.py: 生成 .gni 输出文件

4. 输出生成
   └─> 写入 parts_list.gni、inner_kits.gni、system_kits.gni
```

## 集成点

### 关键文件

| 文件 | 用途 | 输出 |
|------|------|------|
| `generate_targets_gn.py` | GN 文件生成 | 构建系统的 .gni 文件 |
| `load_ohos_build.py` | 解析 ohos.build | 子系统/部件定义 |
| `load_bundle_file.py` | 解析 bundle.json | HPM 组件元数据 |
| `platforms_loader.py` | 加载平台配置 | 平台特定设置 |
| `subsystem_scan.py` | 扫描源代码树 | 子系统发现 |
| `subsystem_info.py` | 子系统注册表 | 元数据查询 |
| `merge_platform_build.py` | 配置合并 | 统一构建配置 |

### 上游依赖
- **resources/config.py**: 全局配置单例
- **resources/global_var.py**: 路径常量
- **util/log_util.py**: 日志记录

### 下游消费者
- **services/loader.py**: 主要消费者
- **GN 构建系统**: 消费生成的 .gni 文件

## 关键技术细节

### Jinja2 模板
使用来自 `third_party/jinja2` 的 Jinja2 进行代码生成：
```python
from jinja2 import Template
```

### 输出文件
生成的文件写入 `out/{product}/build_configs/`：
- `parts_list.gni`: 要构建的部件列表
- `inner_kits.gni`: 内部 API 依赖
- `system_kits.gni`: 系统 API 依赖

### 配置来源
- `build/subsystem_config.json`: 子系统定义
- `vendor/{vendor}/{product}/config.json`: 产品配置
- `{component}/bundle.json`: 组件元数据
- `{component}/ohos.build`: 构建规则

另请参阅：
- [../services/loader.py](../services/loader.py) - 服务包装器
- [../preloader/](../preloader/) - 预构建配置阶段
