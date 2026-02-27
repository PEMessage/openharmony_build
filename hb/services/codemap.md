# HarmonyOS Build System (hb) - Services Layer Codemap

## Table of Contents

1. [Overview](#overview)
2. [Service Architecture](#service-architecture)
3. [Interface Hierarchy](#interface-hierarchy)
4. [Service Implementations](#service-implementations)
5. [Design Patterns](#design-patterns)
6. [Data & Control Flow](#data--control-flow)
7. [Integration Points](#integration-points)
8. [File Reference](#file-reference)

---

## Overview

The `services/` directory contains the core service layer of the HarmonyOS build system (hb). This layer implements the **Service Layer Pattern** and **Facade Pattern** to abstract complex build operations into cohesive, manageable units. Each service encapsulates specific build-phase functionality and coordinates with external tools (GN, Ninja, HPM, HDC).

### Key Responsibilities

- **Build File Generation**: Generate GN build files and configurations (`gn.py`, `loader.py`, `preloader.py`)
- **Build Execution**: Execute compilation via Ninja (`ninja.py`)
- **Package Management**: Handle HPM (HarmonyOS Package Manager) operations (`hpm.py`, `prebuilts.py`)
- **Device Communication**: Manage HDC (HarmonyOS Device Connector) operations (`hdc.py`)
- **Interactive Configuration**: Provide CLI menu interfaces (`menu.py`)
- **SDK Management**: Handle prebuilt SDK operations (`prebuilt_sdk.py`)
- **Independent Build**: Support component-level independent builds (`indep_build.py`)

---

## Service Architecture

### Layered Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                    CLI / Entry Point                             │
├─────────────────────────────────────────────────────────────────┤
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────┐  │
│  │    Menu     │  │   Build     │  │      PrebuiltSdk        │  │
│  │  (menu.py)  │  │  Services   │  │   (prebuilt_sdk.py)     │  │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬─────────────┘  │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │                │                     │                │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌───────────▼────────────┐   │
│  │   Preload   │  │   Loader    │  │   Build Generators     │   │
│  │(preloader)  │  │  (loader)   │  │  (gn, hpm, hdc)        │   │
│  └──────┬──────┘  └──────┬──────┘  └───────────┬────────────┘   │
├─────────┼────────────────┼─────────────────────┼────────────────┤
│         │                │                     │                │
│  ┌──────▼──────┐  ┌──────▼──────┐  ┌───────────▼────────────┐   │
│  │   Build     │  │  Utility    │  │   External Tools       │   │
│  │  Executor   │  │   Layer     │  │ (GN/Ninja/HPM/HDC)     │   │
│  │  (ninja)    │  │             │  │                        │   │
│  └─────────────┘  └─────────────┘  └────────────────────────┘   │
├─────────────────────────────────────────────────────────────────┤
│                      Interface Abstractions                      │
├─────────────────────────────────────────────────────────────────┤
│  ServiceInterface  BuildFileGenerator  BuildExecutor  Load      │
│  PreloadInterface  MenuInterface       PrebuiltSdkInterface     │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interface Hierarchy

### Base Interface: `ServiceInterface`

**File**: `services/interface/service_interface.py`

```python
class ServiceInterface(metaclass=ABCMeta):
    - _args_dict: dict      # Arguments dictionary
    - _exec: str            # Executable path
    + args_dict: property
    + exec: property
    + regist_arg(arg_name: str, arg_value): abstract
    + run(): abstract
```

The root abstraction for all services. Defines the contract for argument registration and execution.

### Specialized Interfaces

#### 1. `BuildFileGeneratorInterface`

**File**: `services/interface/build_file_generator_interface.py`

```python
class BuildFileGeneratorInterface(ServiceInterface):
    - _flags_dict: dict
    + flags_dict: property
    + regist_flag(flag_name: str, flag_value)
    + run(): abstract
```

**Implementations**: `Gn`, `Hpm`, `Hdc`, `IndepBuild`, `PreuiltsService`

Purpose: Services that generate or manipulate build files and configurations. Extends base interface with flag management.

#### 2. `BuildExecutorInterface`

**File**: `services/interface/build_executor_interface.py`

```python
class BuildExecutorInterface(ServiceInterface):
    - _start_time: timestamp
    + regist_arg(arg_name: str, arg_value: str)
    + run(): abstract
```

**Implementations**: `Ninja`

Purpose: Services that execute build processes. Tracks execution start time for metrics.

#### 3. `LoadInterface`

**File**: `services/interface/load_interface.py`

```python
class LoadInterface(ServiceInterface):
    - _config: Config
    + config: property
    + outputs: property
    + __post_init(): abstract
    + run()  # Template Method
    + _generate_*(): abstract methods
    + _check_*(): abstract methods
```

**Implementations**: `OHOSLoader`

Purpose: Template Method pattern for build configuration loading. Defines 17 abstract generation methods.

#### 4. `PreloadInterface`

**File**: `services/interface/preload_interface.py`

```python
class PreloadInterface(ServiceInterface):
    - _config: Config
    + config: property
    + outputs: property
    + __post_init(): abstract
    + run()  # Template Method
    + _generate_*(): abstract methods
```

**Implementations**: `OHOSPreloader`

Purpose: Template Method pattern for preloader phase. Generates pre-build configuration files.

#### 5. `MenuInterface`

**File**: `services/interface/menu_interface.py`

```python
class MenuInterface:
    + select_product(): dict
    + select_compile_option(): dict
```

**Implementations**: `Menu`

Purpose: Abstracts interactive CLI menu operations.

#### 6. `PrebuiltSdkInterface`

**File**: `services/interface/prebuilt_sdk_interface.py`

```python
class PrebuiltSdkInterface(ServiceInterface):
    - _config: Config
    + config: property
    + should_build_sdk(args_dict): bool
    + build_prebuilt_sdk(args_dict): bool
    + _execute_sdk_build(build_args: dict): bool
    + _post_process_sdk(api_version: str): bool
    + _migrate_legacy_sdk(): None
```

**Implementations**: `PrebuiltSdk`

Purpose: Manages SDK building and lifecycle operations.

---

## Service Implementations

### 1. GN Service (`gn.py`)

**Class**: `Gn(BuildFileGeneratorInterface)`

**Responsibility**: Interface to the GN (Generate Ninja) meta-build system.

**Key Components**:

```python
class CMDTYPE(Enum):
    GEN = 1      # Generate build files
    PATH = 2     # Show path between targets
    DESC = 3     # Describe target
    LS = 4       # List targets
    REFS = 5     # Reference tracing
    FORMAT = 6   # Format .gn files
    CLEAN = 7    # Clean build directory
```

**Key Methods**:
- `_regist_gn_path()`: Locates GN executable (platform-aware: x86/aarch64)
- `_execute_gn_gen_cmd()`: Core GN generation with threading animation
- `_convert_args()`: Converts args_dict to GN-compatible format
- `_convert_flags()`: Converts flags_dict to GN flags
- `_check_options_validity()`: Validates command options

**Design Pattern**: **Command Pattern** - Each GN command type is encapsulated as a method executed via `execute_gn_cmd()` dispatcher.

**Threading**: Uses `threading.Event` and `threading.Thread` for animated loading spinner during GN parsing in silent mode.

---

### 2. Ninja Service (`ninja.py`)

**Class**: `Ninja(BuildExecutorInterface)`

**Responsibility**: Execute actual build compilation using Ninja.

**Key Methods**:
- `_regist_ninja_path()`: Locates Ninja executable
- `_execute_ninja_cmd()`: Executes build with environment filtering
- `_convert_args()`: Converts args to Ninja command format

**Environment Management**:
```python
ninja_env = ExecEnviron()
ninja_env.initenv()
ninja_env.allow(ninja_env_allowlist)  # Whitelist-based env filtering
```

**Security Feature**: Uses `compile_env_allowlist.json` to restrict environment variables passed to Ninja.

---

### 3. HPM Service (`hpm.py`)

**Class**: `Hpm(BuildFileGeneratorInterface)`

**Responsibility**: HarmonyOS Package Manager integration for binary dependency management.

**Command Types**:
```python
class CMDTYPE(Enum):
    BUILD = 1
    INSTALL = 2
    PACKAGE = 3
    PUBLISH = 4
    UPDATE = 5
```

**Key Features**:
- **Version Checking**: Background thread checks for HPM updates via npm registry
- **Progress Handling**: Custom line handler for extraction progress (`_custom_line_handle()`)
- **Skip Logic**: Supports `--skip-download`, `--fast-rebuild`, `--local-binarys` flags

**Retry Mechanism**: `max_try=3` for HPM build operations.

---

### 4. HDC Service (`hdc.py`)

**Class**: `Hdc(BuildFileGeneratorInterface)`

**Responsibility**: HarmonyOS Device Connector for device deployment.

**Command Types**:
```python
class CMDTYPE(Enum):
    PUSH = 1           # Push files to device
    LIST_TARGETS = 2   # List connected devices
```

**Deployment Flow**:
1. Mount filesystem as read-write
2. Parse component bundle.json for deployment config
3. Send files via HDC
4. Set file ownership (root:root)
5. Optional device reboot

**Bundle Integration**: Reads `bundle.json` `deployment` section for source/target paths.

---

### 5. Loader Service (`loader.py`)

**Class**: `OHOSLoader(LoadInterface)`

**Responsibility**: Template Method implementation for loading and generating build configuration.

**Template Method Execution** (inherited `run()`):
```python
1. __post_init()              # Initialize configuration
2. _execute_loader_args_display()
3. _check_parts_config_info()
4. _generate_subsystem_configs()
5. _generate_target_platform_parts()
6. _generate_system_capabilities()
7. _generate_stub_targets()
8. _generate_platforms_part_by_src()
9. _generate_target_gn()
10. _generate_phony_targets_build_file()
11. _generate_required_parts_targets()
12. _generate_required_parts_targets_list()
13. _generate_src_flag()
14. _generate_auto_install_part()
15. _generate_platforms_list()
16. _generate_part_different_info()
17. _generate_infos_for_testfwk()
18. _check_product_part_feature()
19. _generate_syscap_files()
20. _cropping_components()
```

**Key Capabilities**:
- Subsystem configuration parsing
- Platform-specific part variant resolution
- Component override mechanism (`_override_one_component()`)
- System capability (syscap) file generation
- Component distribution handling (`_load_component_dist()`)

**Validation**:
- Product part feature validation against whitelist
- Parts config completeness check

---

### 6. Preloader Service (`preloader.py`)

**Class**: `OHOSPreloader(PreloadInterface)`

**Responsibility**: Pre-build configuration generation (Template Method pattern).

**Generated Artifacts**:
| File | Purpose |
|------|---------|
| `platforms.build` | Platform build configuration |
| `build_gnargs.prop` | GN argument properties |
| `features.json` | Part features mapping |
| `syscap.json` | System capabilities |
| `exclusion_modules.json` | Excluded modules |
| `build_config.json` | Build variables |
| `build.prop` | Build properties |
| `parts.json` | Product parts list |
| `parts_config.json` | Parts boolean config |
| `subsystem_config.json` | Subsystem definitions |
| `systemcapability.json` | System capability info |
| `compile_standard_whitelist.json` | Compilation standards |
| `compile_env_allowlist.json` | Environment allowlist |
| `hvigor_compile_hap_whitelist.json` | HAP whitelist |

**OS Level Support**:
- `standard`: Full subsystem config
- `mini`/`small`: Lite subsystem config via `parse_lite_subsystem_config()`

---

### 7. Menu Service (`menu.py`)

**Class**: `Menu(MenuInterface)`

**Responsibility**: Interactive CLI menu using `prompt_toolkit`.

**Components**:
- `InquirerControl`: Custom token list control for selection UI
- `_list_promt()`: List-based prompt handler
- `_question()`: Application assembly with key bindings

**Key Bindings**:
- `Ctrl+Q`/`Ctrl+C`: Cancel
- `Up`/`Down`: Navigation (skips separators/disabled items)
- `Enter`: Selection confirmation

**Selection Flow**:
1. Select OS level (mini/small/standard)
2. Select product (grouped by company)
3. Select compile options per argument

---

### 8. Prebuilt SDK Service (`prebuilt_sdk.py`)

**Class**: `PrebuiltSdk(PrebuiltSdkInterface)`

**Responsibility**: SDK building and management.

**Build Decision Logic**:
```python
should_build_sdk():
  - Skip if --no-prebuilt-sdk=true
  - Skip if product is 'ohos-sdk'
  - Skip if SDK already exists
  - Skip if --prebuilt-sdk=false
```

**Build Pipeline**:
1. `_set_path()`: Configure PATH for ccache and Node.js
2. `_execute_sdk_build()`: Build SDK product with specific GN args
3. `_post_process_sdk()`: Organize output structure
4. `_create_previewer_package()`: Create previewer variant
5. `_migrate_legacy_sdk()`: Migrate legacy SDK-12

**GN Arguments for SDK**:
```python
[
    'skip_generate_module_list_file=true',
    'sdk_platform={platform}',
    'use_cfi=false',
    'use_thin_lto=false',
    'enable_lto_O0=true',
    'sdk_check_flag=false',
    'sdk_for_hap_build=true',
    ...
]
```

---

### 9. Prebuilts Service (`prebuilts.py`)

**Class**: `PreuiltsService(BuildFileGeneratorInterface)`

**Responsibility**: Prebuilt binary dependency management.

**Incremental Update Logic**:
```python
check_whether_need_update():
  - No previous update → Update
  - Part names changed → Update
  - Config files newer than last update → Update
```

**Tracked Files**:
- `build/prebuilts_service/*`
- `build/prebuilts_config.json`
- `build/prebuilts_config.py`
- `build/prebuilts_config.sh`

**State Persistence**: `prebuilts/.local_data/last_update.json`

---

### 10. Independent Build Service (`indep_build.py`)

**Class**: `IndepBuild(BuildFileGeneratorInterface)`

**Responsibility**: Component-level independent building.

**Build Script**: Executes `build/indep_configs/build_indep.sh`

**Features**:
- Local binary cache support (`--local-binarys`)
- Dependency JSON generation (`_generate_dependences_json()`)
- Build type variants: `both`, `onlytest`, `normal`

**Dependency Management**:
- Creates symlinks for local binaries
- Generates `dependences.json` for component mapping

---

## Design Patterns

### 1. Service Layer Pattern

**Implementation**: All services implement `ServiceInterface` or derived interfaces.

**Benefits**:
- Clear separation of concerns
- Consistent argument registration API
- Pluggable service implementations
- Testability through interface mocking

### 2. Template Method Pattern

**Implementation**: `LoadInterface.run()` and `PreloadInterface.run()`

```python
class LoadInterface:
    def run(self):
        self.__post_init__()
        self._generate_subsystem_configs()
        self._generate_target_platform_parts()
        # ... more steps
```

**Benefits**:
- Fixed execution sequence
- Customizable individual steps
- Consistent loader behavior across variants

### 3. Command Pattern

**Implementation**: `Gn.execute_gn_cmd()`, `Hpm.execute_hpm_cmd()`, `Hdc.execute_hdc_cmd()`

```python
def execute_gn_cmd(self, cmd_type: int, **kwargs):
    if cmd_type == CMDTYPE.GEN:
        return self._execute_gn_gen_cmd()
    elif cmd_type == CMDTYPE.PATH:
        return self._execute_gn_path_cmd(**kwargs)
    # ...
```

**Benefits**:
- Encapsulates command execution
- Supports argument variation per command
- Extensible for new command types

### 4. Facade Pattern

**Implementation**: Services provide simplified interfaces to complex subsystems.

Examples:
- `Gn` abstracts GN tool complexity
- `Ninja` abstracts build execution
- `Loader` coordinates multiple utility modules

### 5. Strategy Pattern

**Implementation**: Platform-specific executable resolution.

```python
if sys.platform == "linux" and platform.machine().lower() == "aarch64":
    gn_path = ".../linux-aarch64/bin/gn"
else:
    gn_path = ".../linux-x86/bin/gn"
```

---

## Data & Control Flow

### Build Pipeline Flow

```
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│   Preloader  │────▶│    Loader    │────▶│      GN      │
│  (configure) │     │  (generate)  │     │   (meta)     │
└──────────────┘     └──────────────┘     └──────┬───────┘
                                                  │
                                                  ▼
┌──────────────┐     ┌──────────────┐     ┌──────────────┐
│    HDC       │◀────│    Ninja     │◀────│    HPM       │
│  (deploy)    │     │  (compile)   │     │ (binaries)   │
└──────────────┘     └──────────────┘     └──────────────┘
```

### Preloader Data Flow

```
Product Config ──┐
Subsystem Info ──┼──▶ OHOSPreloader ──▶ JSON/PROP Files ──▶ Loader Input
Build Vars ──────┘    (__post_init__)    (out/preloader/)
```

### Loader Data Flow

```
Subsystem Config ──┐
Platforms Config ──┼──▶ OHOSLoader ──┬──▶ GN Build Files
Parts Info ────────┘   (__post_init__)  ├──▶ JSON Configs
                                          └──▶ Target Lists
```

### Service Execution Flow

```
1. CLI parses arguments
2. Arg.register_args() populates services
3. Service.run() executes
   a. __post_init__() - Initialize state
   b. Execute generation/execution methods
   c. Log results
4. Exception handling via @throw_exception
```

---

## Integration Points

### 1. Utility Layer Dependencies

| Service | Utility Dependencies |
|---------|---------------------|
| `Gn` | `SystemUtil`, `IoUtil`, `LogUtil` |
| `Ninja` | `SystemUtil`, `IoUtil`, `LogUtil` |
| `Hpm` | `SystemUtil`, `ComponentUtil`, `LogUtil` |
| `Hdc` | `SystemUtil`, `IoUtil`, `ComponentUtil` |
| `Loader` | `loader.*`, `file_utils`, `LogUtil` |
| `Preloader` | `IoUtil`, `preloader_process_data`, `LogUtil` |

### 2. Resource Dependencies

| Service | Resource Dependencies |
|---------|----------------------|
| All | `Config`, `Arg` |
| `PrebuiltSdk` | `CURRENT_OHOS_ROOT`, `global_var` |
| `Hpm` | `CURRENT_OHOS_ROOT`, `global_var` |
| `PreuiltsService` | `CURRENT_OHOS_ROOT` |

### 3. External Tool Integration

| Service | External Tool | Version/Path Resolution |
|---------|--------------|------------------------|
| `Gn` | GN | `prebuilts/build-tools/{platform}-{arch}/bin/gn` |
| `Ninja` | Ninja | `prebuilts/build-tools/{platform}-{arch}/bin/ninja` |
| `Hpm` | HPM | `shutil.which("hpm")` or `prebuilts/hpm/node_modules/.bin/hpm` |
| `Hdc` | HDC | `shutil.which("hdc")` |

### 4. Configuration Files

| Service | Input Config Files | Output Files |
|---------|-------------------|--------------|
| `Preloader` | `product_config.json`, `subsystem_config.json` | `out/preloader/{product}/*.json` |
| `Loader` | `out/preloader/{product}/*.json` | `out/{product}/build_configs/*.json` |
| `Gn` | `out/{product}/args.gn` | `out/{product}/build.ninja` |
| `PreuiltsService` | `build/prebuilts_config.json` | `prebuilts/.local_data/last_update.json` |

### 5. Module Dependencies

```
services/
├── gn.py ───────────┬──▶ util/system_util
├── ninja.py ────────┤    util/io_util
├── hpm.py ──────────┤    util/log_util
├── hdc.py ──────────┤    util/component_util
├── loader.py ───────┼──▶ util/loader/*
├── preloader.py ────┤    util/preloader/*
├── menu.py ─────────┤    resources/config
├── prebuilt_sdk.py ─┤    resources/global_var
├── prebuilts.py ────┤    containers/arg
└── indep_build.py ──┘    exceptions/ohos_exception
```

---

## File Reference

### Interface Files

| File | Lines | Purpose |
|------|-------|---------|
| `interface/service_interface.py` | 45 | Base service abstraction |
| `interface/build_file_generator_interface.py` | 42 | Build file generation contract |
| `interface/build_executor_interface.py` | 41 | Build execution contract |
| `interface/load_interface.py` | 145 | Loader template method |
| `interface/preload_interface.py` | 120 | Preloader template method |
| `interface/menu_interface.py` | 30 | Menu abstraction |
| `interface/prebuilt_sdk_interface.py` | 55 | SDK management contract |

### Implementation Files

| File | Lines | Class | Interface | Primary Function |
|------|-------|-------|-----------|------------------|
| `gn.py` | 329 | `Gn` | `BuildFileGeneratorInterface` | GN meta-build orchestration |
| `ninja.py` | 108 | `Ninja` | `BuildExecutorInterface` | Build execution |
| `hpm.py` | 263 | `Hpm` | `BuildFileGeneratorInterface` | Package management |
| `hdc.py` | 138 | `Hdc` | `BuildFileGeneratorInterface` | Device deployment |
| `loader.py` | 996 | `OHOSLoader` | `LoadInterface` | Build config generation |
| `preloader.py` | 348 | `OHOSPreloader` | `PreloadInterface` | Pre-build configuration |
| `menu.py` | 423 | `Menu` | `MenuInterface` | Interactive CLI |
| `prebuilt_sdk.py` | 305 | `PrebuiltSdk` | `PrebuiltSdkInterface` | SDK building |
| `prebuilts.py` | 161 | `PreuiltsService` | `BuildFileGeneratorInterface` | Binary dependency mgmt |
| `indep_build.py` | 152 | `IndepBuild` | `BuildFileGeneratorInterface` | Component builds |

### Total Statistics

- **Total Lines of Code**: ~2,891 (services only)
- **Interface Files**: 7
- **Service Implementations**: 10
- **Design Patterns Used**: 5+

---

## Exception Handling

All services use the `@throw_exception` decorator from `containers.status` for consistent error handling:

```python
@throw_exception
def _execute_gn_gen_cmd(self, **kwargs):
    # Raises OHOSException on failure
```

Error codes are standardized (e.g., '0001' for missing executable, '3001' for unsupported command type).

---

## Conclusion

The services layer implements a well-structured, interface-driven architecture that abstracts the complexity of HarmonyOS build operations. The use of design patterns (Service Layer, Template Method, Command, Facade) ensures maintainability, testability, and extensibility. Clear separation between build file generation (`Gn`, `Hpm`, `Loader`, `Preloader`), build execution (`Ninja`), and utility services (`Hdc`, `Menu`, `PrebuiltSdk`) enables modular development and debugging.
