# services/interface/ Codemap

## Overview

The `services/interface/` directory defines the **Abstract Service Layer** of the HarmonyOS build system (hb). This module establishes the service contracts and abstract base classes (ABCs) that formalize the architectural boundaries between the build system's orchestration logic and its concrete implementations.

---

## 1. Responsibility

This directory contains **Interface Definitions for Services** - pure abstract classes that define the behavioral contracts for all major service components in the build system. Each interface establishes a specific domain boundary within the build pipeline.

### 1.1 Core Service Taxonomy

| Interface | Domain | Purpose |
|-----------|--------|---------|
| `ServiceInterface` | Base | Root abstraction for all services |
| `BuildExecutorInterface` | Execution | Contract for build execution engines (Ninja) |
| `BuildFileGeneratorInterface` | Generation | Contract for build file generators (GN) |
| `LoadInterface` | Loading | Contract for subsystem/part configuration loading |
| `PreloadInterface` | Preloading | Contract for product preloading phase |
| `PrebuiltSdkInterface` | SDK | Contract for prebuilt SDK management |
| `MenuInterface` | UI | Contract for interactive CLI menus |

### 1.2 Architectural Role

```
┌─────────────────────────────────────────────────────────────────┐
│                    Service Interface Layer                      │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────────┐   │
│  │   Service   │ │   Build     │ │   BuildFileGenerator    │   │
│  │  Interface  │ │  Executor   │ │      Interface          │   │
│  │   (Base)    │ │  Interface  │ │                         │   │
│  └──────┬──────┘ └──────┬──────┘ └───────────┬─────────────┘   │
│         │               │                    │                 │
│  ┌──────┴──────┐ ┌──────┴──────┐  ┌──────────┴─────────────┐   │
│  │    Load     │ │   Preload   │  │   PrebuiltSdkInterface │   │
│  │  Interface  │ │  Interface  │  │                        │   │
│  └─────────────┘ └─────────────┘  └────────────────────────┘   │
│                                                                │
│  ┌─────────────────┐                                           │
│  │  MenuInterface  │  (Standalone - no inheritance)            │
│  └─────────────────┘                                           │
└─────────────────────────────────────────────────────────────────┘
```

---

## 2. Design Patterns

### 2.1 Abstract Base Class (ABC) Pattern

All interfaces utilize Python's `abc` module with `ABCMeta` metaclass and `@abstractmethod` decorators to enforce implementation contracts.

```python
# Pattern: Abstract Base Class with Template Method
class ServiceInterface(metaclass=ABCMeta):
    @abstractmethod
    def run(self):
        """Template method - must be implemented by subclasses"""
        pass
```

### 2.2 Template Method Pattern

`LoadInterface` and `PreloadInterface` implement the **Template Method Pattern**, defining a fixed execution sequence while delegating specific steps to concrete implementations:

```python
# LoadInterface.run() - Template Method
def run(self):
    self.__post_init__()                           # Hook
    self._execute_loader_args_display()           # Step 1
    self._check_parts_config_info()               # Step 2
    self._generate_subsystem_configs()            # Step 3
    # ... 18 more steps in fixed sequence
    self._cropping_components()                   # Final step
```

### 2.3 Strategy Pattern

`BuildFileGeneratorInterface` enables **Strategy Pattern** for different build generation backends:
- `Gn` - GN meta-build system
- `IndepBuild` - Independent build system
- `PreuiltsService` - Prebuilt download service

### 2.4 Interface Segregation

Each interface adheres to **Interface Segregation Principle** with focused responsibilities:

| Interface | Method Count | Cohesion |
|-----------|-------------|----------|
| `ServiceInterface` | 2 | Core lifecycle (regist_arg, run) |
| `MenuInterface` | 2 | UI selection (select_product, select_compile_option) |
| `BuildExecutorInterface` | 1 | Execution (run) |
| `BuildFileGeneratorInterface` | 1 | Generation (run) |
| `PrebuiltSdkInterface` | 5 | SDK lifecycle management |

### 2.5 Inheritance Hierarchy

```
ServiceInterface (ABCMeta)
├── BuildExecutorInterface
│   └── Ninja
├── BuildFileGeneratorInterface
│   ├── Gn
│   ├── IndepBuild
│   └── PreuiltsService
├── LoadInterface
│   └── OHOSLoader
├── PreloadInterface
│   └── OHOSPreloader
└── PrebuiltSdkInterface
    └── PrebuiltSdk

MenuInterface (Standalone ABC)
└── Menu
```

---

## 3. Data & Control Flow

### 3.1 Service Contract Specifications

#### 3.1.1 ServiceInterface (Root Contract)

**Location**: `service_interface.py:21`

```python
class ServiceInterface(metaclass=ABCMeta):
    def __init__(self):
        self._args_dict = {}    # Runtime argument registry
        self._exec = ''         # Executable path

    @abstractmethod
    def regist_arg(self, arg_name: str, arg_value):
        """Register build arguments - implementation specific"""
        pass

    @abstractmethod
    def run(self):
        """Execute service - main entry point"""
        pass
```

**Data Flow**:
```
Arg Parsing → regist_arg() → _args_dict → run() → Execution
```

#### 3.1.2 BuildExecutorInterface (Execution Contract)

**Location**: `build_executor_interface.py:25`

| Attribute | Type | Purpose |
|-----------|------|---------|
| `_start_time` | int | Execution timing (Unix timestamp) |
| `_args_dict` | dict | Inherited from ServiceInterface |
| `_exec` | str | Path to executable (ninja) |

**Control Flow**:
```
__init__() → SystemUtil.get_current_time() → _start_time
regist_arg() → Duplicate check → _args_dict[key] = value
run() → Reset _start_time → Execute build
```

#### 3.1.3 BuildFileGeneratorInterface (Generation Contract)

**Location**: `build_file_generator_interface.py:24`

| Attribute | Type | Purpose |
|-----------|------|---------|
| `_flags_dict` | dict | Build flags registry |
| `_args_dict` | dict | Inherited arguments |

**Key Methods**:
- `regist_flag(flag_name, flag_value)` - Register build flags
- `regist_arg(arg_name, arg_value)` - Register arguments (direct assignment, no duplicate check)

#### 3.1.4 LoadInterface (Loading Contract)

**Location**: `load_interface.py:24`

**State Management**:
| Attribute | Type | Source |
|-----------|------|--------|
| `_config` | Config | resources.config.Config singleton |
| `_outputs` | Any | Implementation-defined outputs |
| `_args_dict` | dict | Inherited argument registry |

**Template Method Sequence** (20 steps):

```python
def run(self):
    self.__post_init__()                    # 0. Initialization hook
    self._execute_loader_args_display()     # 1. Logging
    self._check_parts_config_info()         # 2. Validation
    self._generate_subsystem_configs()      # 3. Subsystem configs
    self._generate_target_platform_parts()  # 4. Platform parts
    self._generate_system_capabilities()    # 5. System capabilities
    self._generate_stub_targets()           # 6. Stub targets
    self._generate_platforms_part_by_src()  # 7. Platform parts by source
    self._generate_target_gn()              # 8. GN target files
    self._generate_phony_targets_build_file()  # 9. Phony targets
    self._generate_required_parts_targets()    # 10. Required parts
    self._generate_required_parts_targets_list()  # 11. Parts list
    self._generate_src_flag()               # 12. Source flags
    self._generate_auto_install_part()      # 13. Auto-install
    self._generate_platforms_list()         # 14. Platforms list
    self._generate_part_different_info()    # 15. Part differences
    self._generate_infos_for_testfwk()      # 16. Test framework info
    self._check_product_part_feature()      # 17. Feature validation
    self._generate_syscap_files()           # 18. Syscap files
    self._cropping_components()             # 19. Component cropping
```

#### 3.1.5 PreloadInterface (Preloading Contract)

**Location**: `preload_interface.py:24`

**State Management**:
| Attribute | Type | Purpose |
|-----------|------|---------|
| `_config` | Config | Product configuration |
| `_preloader_outputs` | Any | Generated output artifacts |

**Template Method Sequence** (15 steps):

```python
def run(self):
    self.__post_init__()                           # 0. Initialization
    self._generate_build_prop()                    # 1. Build properties
    self._generate_build_config_json()             # 2. Build config
    self._generate_parts_json()                    # 3. Parts list
    self._generate_parts_config_json()             # 4. Parts config
    self._generate_build_gnargs_prop()             # 5. GN args
    self._generate_features_json()                 # 6. Features
    self._generate_syscap_json()                   # 7. System capabilities
    self._generate_exclusion_modules_json()        # 8. Exclusions
    self._generate_platforms_build()               # 9. Platform build config
    self._generate_subsystem_config_json()         # 10. Subsystem config
    self._generate_systemcapability_json()         # 11. System capability
    self._generate_compile_standard_whitelist_json()  # 12. Compile whitelist
    self._generate_compile_env_allowlist_json()    # 13. Env allowlist
    self._generate_hvigor_compile_whitelist_json() # 14. Hvigor whitelist
```

#### 3.1.6 PrebuiltSdkInterface (SDK Contract)

**Location**: `prebuilt_sdk_interface.py:24`

**Lifecycle Methods**:
| Method | Return | Purpose |
|--------|--------|---------|
| `should_build_sdk(args_dict)` | bool | Predicate for SDK build necessity |
| `build_prebuilt_sdk(args_dict)` | bool | Main SDK build orchestration |
| `_execute_sdk_build(build_args)` | bool | Execute SDK compilation |
| `_post_process_sdk(api_version)` | bool | Post-build artifact processing |
| `_migrate_legacy_sdk()` | None | Legacy SDK migration |

#### 3.1.7 MenuInterface (UI Contract)

**Location**: `menu_interface.py:22`

**Interaction Methods**:
| Method | Return | Purpose |
|--------|--------|---------|
| `select_product()` | dict | Interactive product selection |
| `select_compile_option()` | dict | Interactive build option selection |

---

### 3.2 Data Transformation Flow

```
┌────────────────────────────────────────────────────────────────────────┐
│                        Build Pipeline Flow                              │
├────────────────────────────────────────────────────────────────────────┤
│                                                                        │
│  1. PRELOAD PHASE (PreloadInterface)                                   │
│     ┌──────────────┐                                                   │
│     │ OHOSPreloader│ → Generates: build.prop, parts.json,             │
│     │              │              features.json, syscap.json, ...      │
│     └──────┬───────┘     Output: out/preloader/${product}/            │
│            │                                                           │
│  2. LOAD PHASE (LoadInterface)                                         │
│     ┌──────────────┐                                                   │
│     │  OHOSLoader  │ → Consumes: Preloader outputs                    │
│     │              │ → Generates: BUILD.gn, subsystem configs,        │
│     └──────┬───────┘              parts_targets, system_capabilities   │
│            │                   Output: out/${product}/build_configs/   │
│            │                                                           │
│  3. GENERATION PHASE (BuildFileGeneratorInterface)                     │
│     ┌──────────┐  ┌──────────────┐  ┌──────────────┐                  │
│     │    Gn    │  │  IndepBuild  │  │PreuiltsService│                  │
│     │          │  │              │  │               │                  │
│     └────┬─────┘  └──────┬───────┘  └───────┬───────┘                  │
│          │               │                  │                         │
│          └───────────────┴──────────────────┘                         │
│                          │                                            │
│  4. EXECUTION PHASE (BuildExecutorInterface)                           │
│     ┌──────────┐                                                       │
│     │  Ninja   │ → Consumes: build.ninja (from GN)                    │
│     │          │ → Executes: Parallel build execution                 │
│     └──────────┘                                                       │
│                                                                        │
└────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Integration Points

### 4.1 Concrete Implementations

| Interface | Implementation | File | Responsibility |
|-----------|---------------|------|----------------|
| `BuildExecutorInterface` | `Ninja` | `services/ninja.py:31` | Parallel build execution |
| `BuildFileGeneratorInterface` | `Gn` | `services/gn.py:47` | GN meta-build generation |
| `BuildFileGeneratorInterface` | `IndepBuild` | `services/indep_build.py:30` | Independent component builds |
| `BuildFileGeneratorInterface` | `PreuiltsService` | `services/prebuilts.py:32` | Prebuilt binary downloads |
| `LoadInterface` | `OHOSLoader` | `services/loader.py:35` | Subsystem/part loading |
| `PreloadInterface` | `OHOSPreloader` | `services/preloader.py:28` | Product preloading |
| `PrebuiltSdkInterface` | `PrebuiltSdk` | `services/prebuilt_sdk.py:31` | SDK build management |
| `MenuInterface` | `Menu` | `services/menu.py:45` | Interactive CLI |

### 4.2 Implementation Inheritance Details

#### 4.2.1 Ninja (Build Executor)

```python
class Ninja(BuildExecutorInterface):
    def __init__(self):
        super().__init__()           # Sets _start_time
        self._regist_ninja_path()    # Locates ninja executable
    
    def run(self):
        self._execute_ninja_cmd()    # Override - executes ninja
    
    def _convert_args(self) -> list:
        # Transforms _args_dict to ninja CLI arguments
```

**Integration Points**:
- Consumes: `Config.out_path` (output directory)
- Consumes: `args_dict['build_target']` (targets)
- Consumes: `args_dict['ninja_args']` (additional args)
- Uses: `ExecEnviron` for sandboxed environment

#### 4.2.2 Gn (Build File Generator)

```python
class Gn(BuildFileGeneratorInterface):
    def __init__(self):
        super().__init__()           # Initializes _flags_dict
        self._regist_gn_path()       # Locates gn executable
    
    def run(self):
        self.execute_gn_cmd(CMDTYPE.GEN)
    
    def _convert_args(self) -> list:
        # Transforms _args_dict to --args="..." format
    
    def _convert_flags(self) -> list:
        # Transforms _flags_dict to CLI flags
```

**Integration Points**:
- Consumes: `Config.out_path`, `Config.os_level`
- Consumes: `_args_dict` (converted to `--args=`)
- Consumes: `_flags_dict` (converted to CLI flags)
- Generates: `build.ninja` in output directory

#### 4.2.3 OHOSLoader (Configuration Loader)

```python
class OHOSLoader(LoadInterface):
    def __post_init__(self):
        # Initializes paths from Config
        # Loads platforms info
        # Loads parts config info
    
    def _generate_target_gn(self):
        # Calls generate_targets_gn.gen_targets_gn()
```

**Integration Points**:
- Reads: `out/preloader/${product}/subsystem_config.json`
- Reads: `out/preloader/${product}/platforms.build`
- Writes: `out/${product}/build_configs/`
- Uses: `util/loader/` modules for parsing

#### 4.2.4 OHOSPreloader (Product Preloader)

```python
class OHOSPreloader(PreloadInterface):
    def __post_init__(self):
        self._dirs = Dirs(self._config)
        self._outputs = Outputs(self._dirs.preloader_output_dir)
        self._product = Product(self._dirs, self._config)
```

**Integration Points**:
- Uses: `util/preloader/preloader_process_data.py`
- Reads: Product configuration
- Writes: `out/preloader/${product}/`

### 4.3 External Dependencies

| Interface | External Modules | Purpose |
|-----------|-----------------|---------|
| All | `resources.config.Config` | Global configuration singleton |
| All | `util.log_util.LogUtil` | Structured logging |
| `BuildExecutorInterface` | `util.system_util.SystemUtil` | Command execution |
| `LoadInterface` | `util.loader.*` | Subsystem/part parsing |
| `PreloadInterface` | `util.preloader.*` | Product data processing |
| `PrebuiltSdkInterface` | `resources.global_var.CURRENT_OHOS_ROOT` | Repository root |

### 4.4 File Output Contracts

#### LoadInterface Outputs (to `out/${product}/build_configs/`)

| Method | Output File | Format |
|--------|-------------|--------|
| `_generate_target_platform_parts` | `target_platforms_parts.json` | JSON |
| `_generate_part_different_info` | `parts_different_info.json` | JSON |
| `_generate_platforms_list` | `platforms_list.gni` | GNI |
| `_generate_src_flag` | `parts_src_flag.json` | JSON |
| `_generate_auto_install_part` | `auto_install_parts.json` | JSON |
| `_generate_required_parts_targets` | `required_parts_targets.json` | JSON |
| `_generate_required_parts_targets_list` | `required_parts_targets_list.json` | JSON |
| `_generate_platforms_part_by_src` | `platforms_parts_by_src.json` | JSON |
| `_generate_target_gn` | `subsystem_info/*.gni` | GNI |
| `_generate_phony_targets_build_file` | `phony_target/BUILD.gn` | GN |
| `_generate_stub_targets` | `${platform}-stub/BUILD.gn` | GN |
| `_generate_system_capabilities` | `${platform}_system_capabilities.json` | JSON |
| `_generate_subsystem_configs` | `subsystem_info/*.json` | JSON |
| `_generate_infos_for_testfwk` | `infos_for_testfwk.json` | JSON |
| `_generate_syscap_files` | `system/etc/SystemCapability.json` | JSON |

#### PreloadInterface Outputs (to `out/preloader/${product}/`)

| Method | Output File | Format |
|--------|-------------|--------|
| `_generate_build_prop` | `build.prop` | Properties |
| `_generate_build_config_json` | `build_config.json` | JSON |
| `_generate_parts_json` | `parts.json` | JSON |
| `_generate_parts_config_json` | `parts_config.json` | JSON |
| `_generate_build_gnargs_prop` | `build_gnargs.prop` | Properties |
| `_generate_features_json` | `features.json` | JSON |
| `_generate_syscap_json` | `syscap.json` | JSON |
| `_generate_exclusion_modules_json` | `exclusion_modules.json` | JSON |
| `_generate_platforms_build` | `platforms.build` | JSON |
| `_generate_subsystem_config_json` | `subsystem_config.json` | JSON |
| `_generate_systemcapability_json` | `systemcapability.json` | JSON |
| `_generate_compile_standard_whitelist_json` | `compile_standard_whitelist.json` | JSON |
| `_generate_compile_env_allowlist_json` | `compile_env_allowlist.json` | JSON |
| `_generate_hvigor_compile_whitelist_json` | `hvigor_compile_hap_whitelist.json` | JSON |

---

## 5. Usage Patterns

### 5.1 Service Instantiation

```python
# Pattern 1: Direct instantiation
from services.ninja import Ninja
executor = Ninja()
executor.regist_arg('build_target', ['target1', 'target2'])
executor.run()

# Pattern 2: Factory-based (implied)
from services.gn import Gn
generator = Gn()
generator.regist_arg('product_name', 'product')
generator.regist_flag('gn_flags', ['--ide=json'])
generator.run()
```

### 5.2 Argument Registration Flow

```python
# Duplicate detection with warning (BuildExecutorInterface, LoadInterface, PreloadInterface)
def regist_arg(self, arg_name: str, arg_value: str):
    if arg_name in self._args_dict.keys() and self._args_dict[arg_name] != arg_value:
        LogUtil.hb_warning('duplicated regist arg {}, the original value "{}" will be replace to "{}"'.format(
            arg_name, self._args_dict[arg_name], arg_value))
    self._args_dict[arg_name] = arg_value

# Direct assignment (BuildFileGeneratorInterface)
def regist_arg(self, arg_name: str, arg_value: str):
    self._args_dict[arg_name] = arg_value
```

---

## 6. Summary

The `services/interface/` module establishes a **clean separation of concerns** through abstract base classes, enabling:

1. **Pluggable Architecture**: New build executors, generators, or loaders can be added without modifying orchestration logic
2. **Testability**: Interfaces enable mocking for unit testing
3. **Consistency**: Template methods ensure uniform execution sequences across implementations
4. **Type Safety**: Abstract methods enforce implementation completeness at class definition time

The interface hierarchy maps directly to the build pipeline phases: **Preload → Load → Generate → Execute**, with each phase represented by a focused contract that concrete implementations fulfill.
