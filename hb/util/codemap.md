# util/ - Utility Functions and Helpers

Static utility classes providing cross-cutting functionality for device management, logging, file operations, system commands, and build pipeline stages. All utility classes use the `NoInstance` metaclass to enforce static-only usage.

## Responsibility

**Primary Purpose:** Provide reusable utility functions for the build system across device management, file operations, logging, and build stage processing.

**Key Responsibilities:**
- **Device Management (`device_util.py`)**: HDC (HarmonyOS Device Connector) integration for device communication
- **Logging (`log_util.py`)**: Structured logging with file and console output
- **IO Operations (`io_util.py`)**: File read/write, JSON serialization, subprocess execution
- **System Utilities (`system_util.py`)**: Platform detection, environment variables, path utilities
- **Product Utilities (`product_util.py`)**: Product configuration parsing and validation
- **Type Checking (`type_check_util.py`)**: Argument type validation and conversion
- **Timer Utilities (`timer_util.py`)**: Build phase timing and cost tracking
- **Build Pipeline Stages**:
  - `loader/` - Subsystem and parts loading
  - `preloader/` - Pre-build configuration generation
  - `prebuild/` - Prebuilt binary management
  - `post_build/` - Packaging, signing, and distribution

## Design Patterns

### 1. **Static Utility Classes (NoInstance Pattern)**
All utility classes use the `NoInstance` metaclass from `helper/no_instance.py`:
```python
class LogUtil(metaclass=NoInstance):
    @staticmethod
    def log_info(message):
        ...
```
This prevents accidental instantiation and documents intent.

### 2. **Facade Pattern**
Utility classes provide simplified interfaces to complex operations:
- `LogUtil` abstracts file + console logging
- `IoUtil` wraps file operations with error handling
- `DeviceUtil` provides high-level device commands over HDC

### 3. **Template Method Pattern** (Build Pipeline)
Each build stage subdirectory follows a consistent pattern:
```
preloader/preloader_generate_config.py
preloader/preloader_process_data.py
loader/generate_targets_gn.py
loader/parts_management.py
```

### 4. **Timer Decorator Pattern** (`timer_util.py`)
```python
@TimerUtil.cost_time
@throw_exception
@build_tracker
def run():
    ...
```
Provides cross-cutting timing and metrics collection.

## Data & Control Flow

### Utility Invocation Flow

```
1. Service Layer (services/)
   ├─> DeviceUtil.reboot_device() for HDC commands
   ├─> LogUtil.log_info() for build output
   ├─> IoUtil.read_json() for config files
   └─> SystemUtil.get_platform() for OS detection

2. Module Layer (modules/)
   └─> ProductUtil.get_product_info() for product config

3. Resolver Layer (resolver/)
   └─> TypeCheckUtil.validate() for argument validation

4. Build Pipeline
   ├─> Preloader: Generate build configs
   ├─> Loader: Load subsystem definitions
   ├─> Prebuild: Download dependencies
   └─> Post-build: Package and sign outputs
```

### Pipeline Stage Flow

```
Preloader Stage
├─> preloader_generate_config.py: Generate build_config.json
└─> preloader_process_data.py: Process product/part definitions

Loader Stage
├─> generate_targets_gn.py: Generate GN target files
├─> parts_management.py: Manage component dependencies
└─> subsystem_file.py: Load subsystem configurations

Post-Build Stage
├─> hap_pack.py: Package HAP files
├─> generate_signed_bn.py: Generate signed binaries
└─> package_dist.py: Create distribution packages
```

## Integration Points

### Core Utility Modules

| Module | Key Functions | Consumers |
|--------|--------------|-----------|
| `device_util.py` | `reboot_device()`, `install_hap()`, `shell_command()` | services/hdc.py, modules/* |
| `log_util.py` | `log_info()`, `log_error()`, `init_logger()` | All layers |
| `io_util.py` | `read_json()`, `write_file()`, `run_command()` | services/, util/ |
| `system_util.py` | `get_platform()`, `set_env()` | services/, resolver/ |
| `product_util.py` | `get_product_info()`, `get_product_list()` | services/preloader.py |
| `type_check_util.py` | `validate_type()`, `convert_type()` | resolver/ |
| `timer_util.py` | `cost_time` decorator | modules/ |

### Pipeline Subdirectories

| Directory | Purpose | Key Files |
|-----------|---------|-----------|
| `loader/` | Subsystem loading | `generate_targets_gn.py`, `parts_management.py`, `subsystem_file.py` |
| `preloader/` | Pre-build config | `preloader_generate_config.py`, `preloader_process_data.py` |
| `prebuild/` | Dependency download | `prebuilts_download.py` |
| `post_build/` | Packaging | `hap_pack.py`, `generate_signed_bn.py`, `package_dist.py` |

### Upstream Dependencies
- **helper/**: `NoInstance` metaclass for static utility enforcement
- **resources/**: `Config` singleton for global state, `global_var.py` for paths
- **exceptions/**: `OHOSException` for error handling

### Downstream Consumers
- **services/**: Primary consumer of utilities (HDC, preloader, loader services)
- **modules/**: Use ProductUtil, TimerUtil
- **resolver/**: Use TypeCheckUtil for argument validation
- **main.py**: Uses multiple utilities for orchestration

## Key Files

### Top-Level Utilities
- **`device_util.py`**: HDC device communication wrapper (~100 lines)
- **`log_util.py`**: Structured logging with file/console dual output
- **`io_util.py`**: File I/O operations with JSON support
- **`system_util.py`**: Platform detection and environment management
- **`product_util.py`**: Product configuration parsing
- **`component_util.py`**: Component-level operations
- **`type_check_util.py`**: Runtime type validation
- **`timer_util.py`**: Build timing and metrics
- **`monitor.py`**: Build monitoring utilities
- **`get_target_info.py`**: Target platform information retrieval

### Pipeline Utilities

**loader/** (Subsystem Loading)
- `generate_targets_gn.py`: GN build file generation
- `parts_management.py`: Component dependency management
- `parts_config.py`: Parts configuration parsing
- `platforms_parts.py`: Platform-specific parts loading
- `subsystem_file.py`: Subsystem definition loading

**preloader/** (Pre-Build Configuration)
- `preloader_generate_config.py`: Generate build_config.json
- `preloader_process_data.py`: Process product and subsystem data

**prebuild/** (Dependency Management)
- `prebuilts_download.py`: Download and manage prebuilt binaries

**post_build/** (Packaging and Distribution)
- `hap_pack.py`: HAP (HarmonyOS Ability Package) creation
- `generate_signed_bn.py`: Binary signing utilities
- `package_dist.py`: Distribution package generation
- `package_cc_library.py`: C/C++ library packaging
- `package_entries.py`: Entry point packaging

See also:
- [helper/codemap.md](../helper/codemap.md) - NoInstance metaclass
- [services/codemap.md](../services/codemap.md) - Primary utility consumers
