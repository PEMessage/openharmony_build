# modules/ Directory Codemap

## Overview

The `modules/` directory contains the **Command Execution Layer** of the HarmonyOS build system (hb). This layer implements the **Command Pattern** to encapsulate each hb command (`build`, `clean`, `env`, `set`, `tool`, etc.) as a discrete, self-contained module with well-defined interfaces and execution semantics.

---

## 1. Responsibility

### 1.1 Primary Function
The modules layer serves as the **Command Execution Controller** that:
- **Encapsulates** each hb CLI command as a concrete command object
- **Orchestrates** the multi-phase build lifecycle (preload → load → GN → Ninja)
- **Coordinates** service layer components (Preloader, Loader, BuildFileGenerator, BuildExecutor)
- **Implements** cross-cutting concerns (error handling, logging, monitoring, timing)

### 1.2 Command-to-Module Mapping

| hb Command | Module Class | Interface | Purpose |
|------------|--------------|-----------|---------|
| `hb build` | `OHOSBuildModule` | `BuildModuleInterface` | Full build pipeline (GN + Ninja) |
| `hb clean` | `OHOSCleanModule` | `CleanModuleInterface` | Output directory cleanup |
| `hb env` | `OHOSEnvModule` | `EnvModuleInterface` | Environment setup/validation |
| `hb set` | `OHOSSetModule` | `SetModuleInterface` | Build configuration management |
| `hb tool` | `OHOSToolModule` | `ToolModuleInterface` | GN introspection tools (ls, desc, path, refs) |
| `hb build --indep` | `OHOSIndepBuildModule` | `IndepBuildModuleInterface` | Independent component builds |
| `hb install` | `OHOSInstallModule` | `InstallModuleInterface` | HPM package installation |
| `hb package` | `OHOSPackageModule` | `PackageModuleInterface` | HPM package creation |
| `hb publish` | `OHOSPublishModule` | `PublishModuleInterface` | HPM package publishing |
| `hb update` | `OHOSUpdateModule` | `UpdateModuleInterface` | HPM package updates |
| `hb push` | `OHOSPushModule` | `PushModuleInterface` | Device deployment via HDC |

---

## 2. Design Patterns

### 2.1 Command Pattern (GoF)
Each module implements the Command Pattern:
- **Command Interface**: `ModuleInterface` defines `run()` as the execution contract
- **Concrete Commands**: `OHOSBuildModule`, `OHOSCleanModule`, etc.
- **Invoker**: CLI entry points invoke `module.run()` uniformly
- **Receiver**: Service layer components (`Preloader`, `Loader`, etc.) perform actual work

```
┌─────────────────────────────────────────────────────────────┐
│                     Command Pattern                         │
├─────────────────────────────────────────────────────────────┤
│  ModuleInterface (Abstract Command)                         │
│    └── run() [abstract]                                     │
│         ↑                                                   │
│  ┌──────┴───────────────────────────────────────────────┐   │
│  │ BuildModuleInterface / CleanModuleInterface          │   │
│  │    [Specialized Interfaces]                          │   │
│  │         ↑                                            │   │
│  │  ┌──────┴─────────────────────────────────────────┐  │   │
│  │  │ OHOSBuildModule / OHOSCleanModule             │  │   │
│  │  │    [Concrete Commands]                        │  │   │
│  │  │    └── run() [implementation]                 │  │   │
│  └──┴─────────────────────────────────────────────────┘  │   │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Interface Segregation Principle (ISP)
The module hierarchy demonstrates strict interface segregation:

**Base Interface**: `ModuleInterface`
- Defines common properties: `args_dict`, `args_resolver`
- Defines abstract `run()` method

**Specialized Interfaces**: Each command type extends `ModuleInterface`
- `BuildModuleInterface`: Adds 4 service dependencies + 10 phase methods
- `CleanModuleInterface`: Adds `clean_regular()` and `clean_deep()`
- `ToolModuleInterface`: Adds 6 introspection methods (ls, desc, path, refs, format, clean)
- `EnvModuleInterface`: Adds `env_check()`, `env_install()`, `clean()`
- `SetModuleInterface`: Adds `set_product()`, `set_parameter()`

### 2.3 Template Method Pattern
`BuildModuleInterface` implements the Template Method pattern for build orchestration:

```python
def run(self):
    try:
        self._prebuild_and_preload()  # Template: calls _prebuild() + _preload()
        self._load()
        self._gn()                     # Template: calls _pre/target/post_target_generate()
        self._ninja()                  # Template: calls _pre/target_compilation()
    except OHOSException as exception:
        raise exception
    else:
        self._post_target_compilation()
    finally:
        self._post_build()
```

Hook methods (`_preload()`, `_load()`, etc.) are abstract and implemented by `OHOSBuildModule`.

### 2.4 Singleton Pattern (Anti-pattern usage)
Each module maintains a class-level `_instance` reference with `get_instance()` static method. This enables:
- Global access to the active module instance
- Exception context enrichment with module state
- **Note**: Creates tight coupling; considered an anti-pattern in modern Python

### 2.5 Decorator Pattern (Cross-cutting Concerns)
`OHOSBuildModule` uses decorators for:
- **Performance Tracking**: `@TimerUtil.cost_time` (in interface)
- **Dfx/Metrics**: `@build_tracker` (telemetry collection)
- **Exception Handling**: `@throw_exception` (unified error transformation)

### 2.6 Strategy Pattern (Phase-based Execution)
The `_run_phase()` method implements Strategy pattern for argument resolution:

```python
def _run_phase(self, phase: BuildPhase):
    for phase_arg in [arg for arg in self.args_dict.values() 
                      if arg.arg_phase == phase]:
        self.args_resolver.resolve_arg(phase_arg, self)
```

Arguments register themselves for specific phases; the resolver executes the appropriate strategy per phase.

---

## 3. Data & Control Flow

### 3.1 Build Command Flow (Most Complex)

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        OHOSBuildModule.run()                            │
├─────────────────────────────────────────────────────────────────────────┤
│  Phase 1: PRE_BUILD + PRE_LOAD                                          │
│    ├── _run_phase(BuildPhase.PRE_BUILD)                                 │
│    │      └── args_resolver.resolve_arg() for each PRE_BUILD arg        │
│    └── _preload() [decorated with @build_tracker]                       │
│           ├── _run_phase(BuildPhase.PRE_LOAD)                           │
│           └── preloader.run()  [if not fast_rebuild]                    │
│                                                                         │
│  Phase 2: LOAD                                                          │
│    └── _load() [decorated with @build_tracker]                          │
│           ├── _run_phase(BuildPhase.LOAD)                               │
│           └── loader.run()  [if not fast_rebuild]                       │
│                                                                         │
│  Phase 3: GN (Build File Generation)                                    │
│    └── _gn() [decorated with @TimerUtil.cost_time]                      │
│           ├── _pre_target_generate()                                    │
│           ├── _target_generate() [decorated with @build_tracker]        │
│           │      └── target_generator.run()                             │
│           └── _post_target_generate()                                   │
│                                                                         │
│  Phase 4: NINJA (Compilation)                                           │
│    └── _ninja() [decorated with @TimerUtil.cost_time]                   │
│           ├── _pre_target_compilation()                                 │
│           ├── _target_compilation() [decorated with @build_tracker]     │
│           │      └── target_compiler.run()                              │
│           └── _post_target_compilation() [decorated with @build_tracker]│
│                                                                         │
│  Phase 5: POST_BUILD (Always in finally block)                          │
│    └── _post_build()                                                    │
│                                                                         │
│  Error Handling: Monitor catches exceptions for telemetry               │
└─────────────────────────────────────────────────────────────────────────┘
```

### 3.2 Independent Build Flow (OHOSIndepBuildModule)

```
run()
  └── _target_compilation()
        ├── _rename_buildlog()          # Archive previous build logs
        ├── _run_prebuilts()            # BuildPhase.PRE_BUILD
        ├── _run_hpm()                  # BuildPhase.HPM_DOWNLOAD
        └── _run_indep_build()          # BuildPhase.INDEP_COMPILATION
```

Supports CLI flags: `-i` (source), `-t` (test), or both (`-i -t`)

### 3.3 HPM Package Management Flow

All HPM-related modules (`OHOSInstallModule`, `OHOSPackageModule`, `OHOSPublishModule`, `OHOSUpdateModule`) follow identical pattern:

```
run()
  └── _install() / _package() / _publish() / _update()
        ├── _run_phase()                # Resolve all registered args
        └── hpm.execute_hpm_cmd(CMDTYPE)  # Delegate to HPM service
```

CMDTYPE enum: `INSTALL`, `PACKAGE`, `PUBLISH`, `UPDATE`

### 3.4 Tool Module Flow (Conditional Dispatch)

`OHOSToolModule` implements conditional execution based on which argument is set:

```python
def run(self):
    if self.args_dict['ls'].arg_value:      → list_targets()
    elif self.args_dict['desc'].arg_value:  → desc_targets()
    elif self.args_dict['path'].arg_value:  → path_targets()
    elif self.args_dict['refs'].arg_value:  → refs_targets()
    elif self.args_dict['format'].arg_value:→ format_targets()
    elif self.args_dict['clean'].arg_value: → clean_targets()
    else:                                   → Arg.get_help(ModuleType.TOOL)
```

---

## 4. Integration Points

### 4.1 Upstream Dependencies (Consumers)

| Consumer | Relationship | Purpose |
|----------|--------------|---------|
| `hb` CLI entry point | Invoker | Parses argv and dispatches to module.run() |
| `containers.arg` | Phase/Type enums | `BuildPhase`, `CleanPhase`, `ModuleType` |
| `resolver.interface.args_resolver_interface` | Dependency Injection | `ArgsResolverInterface` resolves arguments per phase |
| `services.interface.*` | Service Layer | Preloader, Loader, BuildFileGenerator, BuildExecutor, MenuInterface |
| `services.hpm` | HPM Integration | `CMDTYPE` enum, `execute_hpm_cmd()` |
| `services.hdc` | Device Integration | `CMDTYPE` enum, `execute_hdc_cmd()` |
| `util.log_util` | Logging | `LogUtil.hb_info()`, `LogUtil.hb_error()` |
| `util.monitor` | Telemetry | `Monitor.run()` for error telemetry |
| `dfx.build_tracker` | Metrics | `@build_tracker` decorator |
| `exceptions.ohos_exception` | Error Handling | `OHOSException` for domain errors |

### 4.2 Downstream Dependencies (Services Used)

```
┌─────────────────────────────────────────────────────────────────┐
│                     Service Dependencies                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  BuildModuleInterface                                           │
│    ├── PreloadInterface        [preloader]  - Parse build args  │
│    ├── LoadInterface           [loader]     - Load build config │
│    ├── BuildFileGeneratorInterface [target_generator] - GN gen  │
│    └── BuildExecutorInterface  [target_compiler] - Ninja build  │
│                                                                 │
│  IndepBuildModuleInterface                                      │
│    ├── BuildFileGeneratorInterface [prebuilts]                  │
│    ├── BuildFileGeneratorInterface [hpm]                        │
│    └── BuildFileGeneratorInterface [indep_build]                │
│                                                                 │
│  HPM Modules (Install/Package/Publish/Update)                   │
│    └── BuildFileGeneratorInterface [hpm]                        │
│                                                                 │
│  PushModule                                                     │
│    └── BuildFileGeneratorInterface [hdc]                        │
│                                                                 │
│  SetModule                                                      │
│    └── MenuInterface [menu]                                     │
│                                                                 │
│  ToolModule                                                     │
│    └── BuildFileGeneratorInterface [gn]                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.3 Dependency Injection Pattern

Modules receive dependencies via constructor injection:

```python
class OHOSBuildModule(BuildModuleInterface):
    def __init__(self,
                 args_dict: dict,                    # CLI arguments
                 args_resolver: ArgsResolverInterface, # Phase resolver
                 preloader: PreloadInterface,        # Service 1
                 loader: LoadInterface,              # Service 2
                 target_generator: BuildFileGeneratorInterface,  # Service 3
                 target_compiler: BuildExecutorInterface):       # Service 4
```

This enables:
- **Testability**: Mock implementations can be injected
- **Flexibility**: Different service implementations per context
- **Separation of Concerns**: Modules orchestrate, services execute

---

## 5. Class Hierarchy

```
abc.ABC (Python stdlib)
  └── ModuleInterface [ABCMeta]
        ├── BuildModuleInterface
        │     └── OHOSBuildModule
        │
        ├── CleanModuleInterface
        │     └── OHOSCleanModule
        │
        ├── EnvModuleInterface
        │     └── OHOSEnvModule
        │
        ├── SetModuleInterface
        │     └── OHOSSetModule
        │
        ├── ToolModuleInterface
        │     └── OHOSToolModule
        │
        ├── IndepBuildModuleInterface
        │     └── OHOSIndepBuildModule
        │
        ├── InstallModuleInterface
        │     └── OHOSInstallModule
        │
        ├── PackageModuleInterface
        │     └── OHOSPackageModule
        │
        ├── PublishModuleInterface
        │     └── OHOSPublishModule
        │
        ├── UpdateModuleInterface
        │     └── OHOSUpdateModule
        │
        └── PushModuleInterface
              └── OHOSPushModule
```

---

## 6. File Inventory

### 6.1 Concrete Implementations (Root Directory)

| File | Lines | Purpose |
|------|-------|---------|
| `ohos_build_module.py` | 154 | Full build pipeline with 9 phases |
| `ohos_clean_module.py` | 53 | Regular and deep clean operations |
| `ohos_env_module.py` | 48 | Environment check and install |
| `ohos_set_module.py` | 50 | Product/parameter configuration |
| `ohos_tool_module.py` | 62 | GN introspection tool wrapper |
| `ohos_indep_build_module.py` | 140 | Independent component builds |
| `ohos_install_module.py` | 64 | HPM package installation |
| `ohos_package_module.py` | 62 | HPM package creation |
| `ohos_publish_module.py` | 62 | HPM package publishing |
| `ohos_update_module.py` | 62 | HPM package updates |
| `ohos_push_module.py` | 66 | HDC device deployment |

### 6.2 Interface Definitions (interface/ Subdirectory)

| File | Lines | Purpose |
|------|-------|---------|
| `module_interface.py` | 39 | Base ABC for all modules |
| `build_module_interface.py` | 133 | Template method for build phases |
| `clean_module_interface.py` | 39 | Clean operation contract |
| `env_module_interface.py` | 44 | Environment operation contract |
| `set_module_interface.py` | 40 | Configuration operation contract |
| `tool_module_interface.py` | 69 | Tool dispatch contract |
| `indep_build_module_interface.py` | 38 | Independent build contract |
| `install_module_interface.py` | 38 | Install operation contract |
| `package_module_interface.py` | 38 | Package operation contract |
| `publish_module_interface.py` | 38 | Publish operation contract |
| `update_module_interface.py` | 38 | Update operation contract |
| `push_module_interface.py` | 38 | Push operation contract |

---

## 7. Key Implementation Details

### 7.1 Build Phase Enumeration

```python
class BuildPhase():
    NONE = 0
    PRE_BUILD = 1
    PRE_LOAD = 2
    LOAD = 3
    PRE_TARGET_GENERATE = 4
    TARGET_GENERATE = 5      # GN phase
    POST_TARGET_GENERATE = 6
    PRE_TARGET_COMPILATION = 7
    TARGET_COMPILATION = 8   # Ninja phase
    POST_TARGET_COMPILATION = 9
    POST_BUILD = 10
    HPM_DOWNLOAD = 11
    INDEP_COMPILATION = 12
```

### 7.2 Fast Rebuild Optimization

`OHOSBuildModule` supports fast rebuild mode that skips expensive phases:

```python
def _preload(self):
    if self.args_dict.get('fast_rebuild') and \
       not self.args_dict.get('fast_rebuild').arg_value:
        self.preloader.run()  # Skip if fast_rebuild=True
```

### 7.3 Exception Propagation

All modules follow consistent exception handling:
- Catch `OHOSException` from service layer
- Re-raise for upstream handling
- Execute cleanup in `finally` blocks

---

## 8. Architectural Strengths

1. **Extensibility**: New commands add new `*ModuleInterface` + `OHOS*Module` pair
2. **Testability**: Dependency injection enables mocking
3. **Observability**: Decorators provide uniform telemetry
4. **Consistency**: All modules follow same structural pattern
5. **Separation**: Interface segregation prevents fat interfaces

## 9. Potential Improvements

1. **Singleton Anti-pattern**: Remove `_instance` references; use dependency injection
2. **Code Duplication**: `_run_phase()` implementations are nearly identical
3. **Mixed Abstractions**: `ToolModuleInterface` mixes concerns (ls, desc, format, clean)
4. **Documentation**: Phase semantics (PRE_BUILD vs PRE_LOAD) lack detailed documentation
5. **Error Codes**: All exceptions use generic code `'0000'`

---

*Generated: HarmonyOS Build System (hb) Modules Analysis*
