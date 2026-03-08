# util/post_build/ - 构建后处理

用于分析构建输出、计算ROM统计信息和在编译完成后生成报告的构建后工具。

## 职责

**主要目的：** 构建后分析和报告工具。

**关键职责：**
- **ROM统计 (`part_rom_statistics.py`)**：从构建产物计算各部件的ROM使用情况

## 设计模式

### 1. **分析器模式**
`part_rom_statistics.py` 分析ELF二进制文件和map文件，计算每个组件的代码/数据大小。

### 2. **报告器模式**
以JSON/文本格式生成结构化报告，用于构建指标。

## 数据和控制流

### 构建后分析流程

```
1. 构建完成
   └─> Ninja构建结束

2. 统计信息收集
   └─> part_rom_statistics.py 分析:
       ├─> ELF二进制文件大小
       ├─> Map文件分析
       └─> 各部件细分

3. 报告生成
   └─> 输出JSON/文本报告到 out/{product}/images/
```

## 集成点

### 关键文件

| 文件 | 用途 | 输出 |
|------|------|------|
| `part_rom_statistics.py` | ROM大小分析 | ROM统计报告 |

### 上游依赖
- **resources/config.py**: 获取产品/构建信息
- **util/log_util.py**: 日志记录

### 下游消费者
- **services/loader.py**: 可能调用构建后分析
- **CI/CD系统**: 消费生成的报告

### 输入文件
- `out/{product}/images/*.elf`: 编译后的二进制文件
- `out/{product}/images/*.map`: 链接器map文件

### 输出文件
- `out/{product}/images/rom_statistics.json`: 各部件大小细分

## 关键技术细节

### 大小计算
分析ELF段：
```python
# .text, .rodata, .data, .bss 段
# 通过符号表按部件归因
```

### 报告格式
```json
{
  "part_name": {
    "text_size": 1234,
    "rodata_size": 567,
    "data_size": 89,
    "bss_size": 0
  }
}
```

另请参阅：
- [../loader/](../loader/) - 构建加载阶段
- [../preloader/](../preloader/) - 构建前配置阶段
