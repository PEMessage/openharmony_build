# util/preloader/ - 预构建配置

用于解析产品配置、生成 build_config.json 并准备构建环境的预加载工具。这是构建流水线的第一阶段。

## 职责

**主要目的：** 解析产品和子系统配置以生成构建配置文件。

**关键职责：**
- **Vendor/产品配置解析 (`parse_vendor_product_config.py`)**: 解析 vendor 和产品 JSON 配置
- **轻量子系统解析 (`parse_lite_subsystems_config.py`)**: 解析轻量化设备的子系统配置
- **数据处理 (`preloader_process_data.py`)**: 处理并合并所有配置数据
- **构建配置生成**: 为下游阶段生成 build_config.json

## 设计模式

### 1. **解析器链模式**
多个专用解析器用于不同的配置类型：
```
parse_vendor_product_config.py → Vendor/产品设置
parse_lite_subsystems_config.py → 轻量子系统定义
preloader_process_data.py → 合并并验证所有数据
```

### 2. **配置构建器模式**
`preloader_process_data.py` 作为构建器，从多个来源组装配置：
- 产品配置（特性、子系统）
- Vendor 配置（硬件设置）
- 子系统配置（组件定义）
- 平台配置（开发板特定设置）

### 3. **验证模式**
输出生成前的配置验证：
- 检查必需字段是否存在
- 验证子系统/部件存在性
- 验证特性依赖关系

## 数据与控制流

### 预加载器流水线流程

```
1. 服务调用
   └─> services/preloader.py 调用预加载器工具

2. 配置解析
   ├─> parse_vendor_product_config.py: 解析 vendor/{vendor}/{product}/config.json
   ├─> parse_lite_subsystems_config.py: 解析轻量子系统定义
   └─> 加载 build/subsystem_config.json

3. 数据处理
   └─> preloader_process_data.py:
       ├─> 合并产品 + vendor 配置
       ├─> 解析子系统依赖
       ├─> 处理特性标志
       └─> 验证配置

4. 输出生成
   └─> 将 build_config.json 写入 out/{product}/build_configs/
```

## 集成点

### 关键文件

| 文件 | 用途 | 输出 |
|------|------|------|
| `parse_vendor_product_config.py` | 解析产品配置 | 产品设置字典 |
| `parse_lite_subsystems_config.py` | 解析轻量子系统 | 子系统定义 |
| `preloader_process_data.py` | 合并并验证 | build_config.json |

### 上游依赖
- **resources/config.py**: 全局配置（产品、变体）
- **resources/global_var.py**: 路径常量
- **util/log_util.py**: 日志记录
- **util/io_util.py**: 文件操作

### 下游消费者
- **services/preloader.py**: 主要消费者
- **util/loader/**: 使用生成的 build_config.json
- **services/loader.py**: 读取 build_config.json

### 输入文件
- `vendor/{vendor}/{product}/config.json`: 产品配置
- `vendor/{vendor}/{product}/subsystem_config.json`: 产品子系统
- `build/subsystem_config.json`: 全局子系统定义
- `build/lite/components/*.json`: 轻量组件定义

### 输出文件
- `out/{product}/build_configs/build_config.json`: 生成的构建配置

### build_config.json 结构
```json
{
  "product_name": "rk3568",
  "device_company": "rockchip",
  "target_cpu": "arm",
  "subsystems": [
    {"subsystem": "aafwk", "components": [...]}
  ],
  "features": {...}
}
```

## 关键技术细节

### 配置优先级
1. 产品配置（最高优先级）
2. Vendor 配置
3. 平台配置
4. 默认子系统配置（最低优先级）

### 依赖解析
- 从 ohos.build 文件解析组件依赖
- 构建顺序的拓扑排序
- 特性标志评估

另请参阅：
- [../loader/](../loader/) - 流水线的下一阶段
- [../../services/preloader.py](../../services/preloader.py) - 服务包装器
- [../../resources/config.py](../../resources/config.py) - 全局配置
