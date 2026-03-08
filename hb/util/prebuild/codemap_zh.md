# util/prebuild/ - 预构建设置

用于管理预构建二进制文件、应用补丁以及在编译前准备构建环境的预构建工具。

## 职责

**主要用途:** 预构建环境准备和依赖管理。

**关键职责:**
- **补丁处理 (`patch_process.py`)**: 将补丁应用到源代码或预构建组件

## 设计模式

### 1. **补丁应用模式**
`patch_process.py` 管理源代码的补丁应用：
- 将 `.patch` 文件应用到源目录
- 跟踪已应用的补丁以避免重复应用
- 处理补丁冲突和失败

## 数据与控制流

### 预构建流程

```
1. 构建初始化
   └─> services/preloader.py 或预构建阶段启动

2. 补丁应用 (如需要)
   └─> patch_process.py:
       ├─> 扫描补丁文件
       ├─> 验证补丁可应用性
       └─> 将补丁应用到源代码树

3. 环境就绪
   └─> 继续到预加载器/加载器阶段
```

## 集成点

### 关键文件

| 文件 | 用途 | 输入/输出 |
|------|------|--------------|
| `patch_process.py` | 补丁管理 | .patch 文件 → 修改后的源代码 |

### 上游依赖
- **resources/global_var.py**: 路径常量
- **util/log_util.py**: 日志记录
- **util/io_util.py**: 文件操作

### 下游消费者
- **services/preloader.py**: 可能调用补丁应用
- **services/prebuilts.py**: 下载和补丁预构建文件

### 输入文件
- `build/patches/*.patch`: 补丁文件
- `.applied_patches`: 跟踪文件

## 关键技术细节

### 补丁应用
```python
# 检查补丁是否已应用
# 验证补丁是否可以干净地应用
# 使用 git apply 或 patch 命令应用
# 在跟踪文件中记录
```

另请参阅:
- [../preloader/](../preloader/) - 配置生成
- [../../services/prebuilts.py](../../services/prebuilts.py) - 预构建管理
