# hb/ - HarmonyOS Build System (hb)

HarmonyOS command-line build system implementing a layered architecture with Command Pattern, Template Method, and Dependency Injection. Supports both full source builds and independent component builds via HPM (HarmonyOS Package Manager).

## Responsibility

**Primary Purpose:** CLI entry point and orchestration layer for the HarmonyOS build system.

**Key Responsibilities:**
- **Entry Point (`hb/__main__.py`)**: Bootstrapper that validates OHOS workspace and dynamically loads `main.py`
- **Orchestrator (`main.py`)**: Central command dispatcher and module initializer
- **Module Factory**: Creates and configures 11 command modules (build, set, clean, env, tool, indep_build, install, package, publish, update, push)
- **Dependency Injection Container**: Wires services, resolvers, and modules together
- **Path Management**: Sets up PATH for prebuilt tools (HPM, Node.js)

## Architecture Overview

### Layered Architecture

```
┌─────────────────────────────────────────────────────────────┐
│  CLI Entry (hb/__main__.py)                                 │
├─────────────────────────────────────────────────────────────┤
│  Orchestrator (main.py) - Command Dispatch                  │
├─────────────────────────────────────────────────────────────┤
│  Modules Layer (modules/) - Command Execution               │
├─────────────────────────────────────────────────────────────┤
│  Resolver Layer (resolver/) - Argument Parsing              │
├─────────────────────────────────────────────────────────────┤
│  Services Layer (services/) - Build Operations              │
├─────────────────────────────────────────────────────────────┤
│  Utilities Layer (util/) - Helper Functions                 │
├─────────────────────────────────────────────────────────────┤
│  Infrastructure Layer (containers/, resources/, helper/)    │
└─────────────────────────────────────────────────────────────┘
```

### Core Components in Root

| File | Purpose | Key Classes/Functions |
|------|---------|----------------------|
| `__main__.py` | Entry point bootstrapper | `is_in_ohos_dir()`, dynamic import of main.py |
| `main.py` | Orchestrator and DI container | `Main` class, module initializers, `_is_indep_build()` |
| `setup.py` | Python package setup | Package metadata, entry_points for `hb` command |

## Directory Structure

### 1. containers/ - Value Objects and Enums

**Purpose:** Data containers, enumerations, and decorators for the build system.

| File | Responsibility |
|------|----------------|
| `arg.py` | Core argument system: `Arg` class, `ModuleType` enum, `BuildPhase` enum |
| `status.py` | Exception handling decorator `@throw_exception`, `judge_indep()` function |
| `colors.py` | ANSI color codes for terminal output |

**Key Design Patterns:**
- **Enum Pattern:** `ModuleType`, `BuildPhase`, `ArgType`, `CleanPhase` for type-safe constants
- **Value Object Pattern:** `Arg` class encapsulates argument metadata (name, type, phase, resolver)
- **Decorator Pattern:** `@throw_exception` for centralized exception handling

### 2. exceptions/ - Exception Hierarchy

**Purpose:** Custom exception types with error code taxonomy.

| File | Responsibility |
|------|----------------|
| `ohos_exception.py` | `OHOSException` class with error code lookup from `status.json` |

**Error Code Taxonomy:**
- First digit indicates build stage: '1'=preloader, '2'=loader, '3'=GN, '4'=ninja
- Codes map to solutions in `resources/status/status.json`

### 3. helper/ - Metaclasses and Utilities

**Purpose:** Metaclasses for enforcing design constraints.

| File | Responsibility |
|------|----------------|
| `singleton.py` | `Singleton` metaclass for single-instance classes |
| `no_instance.py` | `NoInstance` metaclass for static-only classes |
| `separator.py` | `Separator` class for menu grouping |

**Key Design Patterns:**
- **Singleton Pattern:** Config class uses Singleton for global state
- **Static Class Pattern:** LogUtil, IoUtil use NoInstance to prevent instantiation

### 4. modules/ - Command Execution Layer

**Purpose:** Implement command execution for each hb subcommand.

#### Interface Files (modules/interface/)
| File | Responsibility |
|------|----------------|
| `module_interface.py` | Base `ModuleInterface` - all modules extend this |
| `build_module_interface.py` | `BuildModuleInterface` with Template Method pattern for build phases |
| `set_module_interface.py` | Interface for `hb set` command |
| `clean_module_interface.py` | Interface for `hb clean` command |
| `env_module_interface.py` | Interface for `hb env` command |
| `tool_module_interface.py` | Interface for `hb tool` command |
| `indep_build_module_interface.py` | Interface for independent build |
| `install_module_interface.py` | Interface for `hb install` command |
| `package_module_interface.py` | Interface for `hb package` command |
| `publish_module_interface.py` | Interface for `hb publish` command |
| `update_module_interface.py` | Interface for `hb update` command |
| `push_module_interface.py` | Interface for `hb push` command |

#### Implementation Files
| File | Responsibility | Key Methods |
|------|----------------|-------------|
| `ohos_build_module.py` | Standard build execution | `_preload()`, `_load()`, `_target_generate()`, `_target_compilation()` |
| `ohos_indep_build_module.py` | Independent component build | `_run_prebuilts()`, `_run_hpm()`, `_run_indep_build()` |
| `ohos_set_module.py` | Product/configuration selection | `set_product()`, `set_parameter()` |
| `ohos_clean_module.py` | Build output cleanup | - |
| `ohos_env_module.py` | Environment setup/check | - |
| `ohos_tool_module.py` | GN tool commands | - |
| `ohos_install_module.py` | HPM package install | - |
| `ohos_package_module.py` | HPM package creation | - |
| `ohos_publish_module.py` | HPM package publish | - |
| `ohos_update_module.py` | HPM package update | - |
| `ohos_push_module.py` | Push to device via HDC | - |

### 5. resolver/ - Argument Resolution Layer

**Purpose:** Parse CLI arguments and resolve them to build configuration.

#### Interface
| File | Responsibility |
|------|----------------|
| `interface/args_resolver_interface.py` | `ArgsResolverInterface` base class with `resolve_arg()` method |

#### Resolvers
| File | Responsibility | Key Methods |
|------|----------------|-------------|
| `build_args_resolver.py` | Resolves build arguments | `resolve_product()`, `resolve_ccache()`, `resolve_gn_args()`, `resolve_build_target()` |
| `set_args_resolver.py` | Resolves set arguments | `resolve_product_name()`, `resolve_all()` |
| `clean_args_resolver.py` | Resolves clean arguments | - |
| `env_args_resolver.py` | Resolves env arguments | - |
| `tool_args_resolver.py` | Resolves tool arguments | - |
| `indep_build_args_resolver.py` | Resolves independent build args | `get_part_name()` |
| `install_args_resolver.py` | Resolves install arguments | - |
| `package_args_resolver.py` | Resolves package arguments | - |
| `publish_args_resolver.py` | Resolves publish arguments | - |
| `update_args_resolver.py` | Resolves update arguments | - |
| `push_args_resolver.py` | Resolves push arguments | - |
| `judge_indep_args_resolver.py` | Determines if build is independent | `is_indep_args()` |

#### Factory
| File | Responsibility |
|------|----------------|
| `args_factory.py` | Creates argparse options from JSON definitions |

### 6. services/ - Core Build Services

**Purpose:** Encapsulate build operations and external tool invocations.

#### Interface Files (services/interface/)
| File | Responsibility |
|------|----------------|
| `service_interface.py` | Base `ServiceInterface` with `args_dict`, `exec` properties |
| `preload_interface.py` | Interface for preloader service |
| `load_interface.py` | Interface for loader service |
| `build_file_generator_interface.py` | Interface for GN/HPM build file generation |
| `build_executor_interface.py` | Interface for Ninja build execution |
| `menu_interface.py` | Interface for interactive menu |
| `prebuilt_sdk_interface.py` | Interface for prebuilt SDK handling |

#### Service Implementations
| File | Responsibility | Key Methods |
|------|----------------|-------------|
| `preloader.py` | Preload product configuration | `_generate_platforms_build()`, `_generate_features_json()`, `_generate_parts_json()`, `_generate_build_config_json()` |
| `loader.py` | Load subsystem/part information | `_check_args()`, `_generate_target_gn()`, `_generate_syscap_files()`, `_generate_system_capabilities()` |
| `gn.py` | GN build file generation | `execute_gn_cmd()`, `_execute_gn_gen_cmd()`, `_execute_gn_path_cmd()`, `_execute_gn_desc_cmd()` |
| `ninja.py` | Ninja build execution | `_execute_ninja_cmd()`, `_regist_ninja_path()` |
| `hpm.py` | HPM (HarmonyOS Package Manager) integration | `execute_hpm_cmd()`, `_execute_hpm_build_cmd()`, `_execute_hpm_install_cmd()` |
| `indep_build.py` | Independent build orchestration | `run()`, `_convert_flags()`, `_generate_dependences_json()` |
| `prebuilts.py` | Prebuilt binary handling | - |
| `prebuilt_sdk.py` | Prebuilt SDK compilation | `should_build_sdk()`, `run()` |
| `menu.py` | Interactive product selection | `select_product()`, `select_compile_option()`, `_list_promt()` |
| `hdc.py` | HDC (Huawei Debug Client) integration | - |

### 7. util/ - Utility Functions

**Purpose:** Helper utilities organized by functional area.

#### Root Utils
| File | Responsibility |
|------|----------------|
| `log_util.py` | Logging utilities: `LogUtil.hb_info()`, `hb_warning()`, `hb_error()` |
| `system_util.py` | System operations: `SystemUtil.exec_command()`, `get_current_time()` |
| `io_util.py` | File I/O: `IoUtil.read_json_file()`, `dump_json_file()` |
| `type_check_util.py` | Type validation utilities |
| `component_util.py` | Component/bundle operations |
| `product_util.py` | Product discovery and management |
| `device_util.py` | Device operations |
| `timer_util.py` | Timing decorators |
| `monitor.py` | Build monitoring |

#### Loader Utils (util/loader/)
| File | Responsibility |
|------|----------------|
| `load_ohos_build.py` | Parse ohos.build files |
| `load_bundle_file.py` | Parse bundle.json files |
| `subsystem_info.py` | Subsystem information management |
| `subsystem_scan.py` | Subsystem scanning |
| `generate_targets_gn.py` | Generate BUILD.gn files |
| `platforms_loader.py` | Platform configuration loading |
| `merge_platform_build.py` | Platform build merging |

#### Preloader Utils (util/preloader/)
| File | Responsibility |
|------|----------------|
| `preloader_process_data.py` | Data processing for preloader |
| `parse_lite_subsystems_config.py` | Parse lite subsystem configs |
| `parse_vendor_product_config.py` | Parse vendor/product configs |

#### Prebuild Utils (util/prebuild/)
| File | Responsibility |
|------|----------------|
| `patch_process.py` | Patch application |

#### Post-build Utils (util/post_build/)
| File | Responsibility |
|------|----------------|
| `part_rom_statistics.py` | ROM size statistics |

### 8. resources/ - Configuration and Data

**Purpose:** Configuration files, JSON schemas, and argument definitions.

| File/Directory | Responsibility |
|----------------|----------------|
| `config.py` | `Config` singleton - central configuration management |
| `global_var.py` | Global constants: paths, filenames, VERSION |
| `args/default/*.json` | Default argument definitions for each command |
| `config/config.json` | Default build configuration template |
| `status/status.json` | Error code definitions and solutions |

**Argument JSON Files:**
- `buildargs.json` - Build command arguments
- `setargs.json` - Set command arguments
- `cleanargs.json` - Clean command arguments
- `envargs.json` - Env command arguments
- `toolargs.json` - Tool command arguments
- `indepbuildargs.json` - Independent build arguments
- `installargs.json` - Install command arguments
- `packageargs.json` - Package command arguments
- `publishargs.json` - Publish command arguments
- `updateargs.json` - Update command arguments
- `pushargs.json` - Push command arguments

### 9. test/ - Unit Tests

**Purpose:** Unit tests for services.

| File | Responsibility |
|------|----------------|
| `unitTest/services/preloader_test.py` | Preloader unit tests |
| `unitTest/services/loader_test.py` | Loader unit tests |
| `unitTest/services/test.py` | General tests |

## Design Patterns

### 1. **Facade Pattern** (`__main__.py`)
Simplifies entry point by hiding workspace validation and dynamic module loading complexity.

### 2. **Factory Method Pattern** (`main.py`)
Each command has a dedicated factory method:
- `_init_build_module()` - Standard build with preloader→loader→GN→Ninja pipeline
- `_init_indep_build_module()` - Independent build with HPM integration
- `_init_tool_module()` - Single tool execution

### 3. **Dependency Injection**
Services injected via constructors rather than instantiated internally:
```python
preloader = OHOSPreloader()
loader = OHOSLoader()
generate_ninja = Gn()
ninja = Ninja()
return OHOSBuildModule(args_dict, resolver, preloader, loader, generate_ninja, ninja)
```

### 4. **Strategy Pattern** (Module Selection)
`module_initializers` dictionary maps commands to factory methods:
```python
module_initializers = {
    'build': main._init_indep_build_module if main._is_indep_build() else main._init_build_module,
    'set': main._init_set_module,
    'clean': main._init_clean_module,
    # ... 11 commands total
}
```

### 5. **Template Method Pattern** (`BuildModuleInterface`)
Defines build phase sequence:
```python
def run(self):
    self._prebuild_and_preload()
    self._load()
    self._gn()
    self._ninja()
    self._post_target_compilation()
    self._post_build()
```

### 6. **Singleton Pattern** (`helper/singleton.py`)
`Config` class ensures single global configuration instance.

### 7. **Decorator Chain**
`@throw_exception` + `@build_tracker` provide cross-cutting concerns (error handling, metrics).

## Data & Control Flow

### Standard Build Flow

```
1. CLI Invocation
   └─> hb build --product rk3568

2. Entry Validation (__main__.py)
   └─> is_in_ohos_dir() - Walk up directory tree looking for marker files

3. Orchestration (main.py::Main.main())
   ├─> Parse arguments (Arg.parse_all_args)
   ├─> Prebuild SDK check (_prebuild_ohos_sdk)
   ├─> Module selection via module_initializers dict
   └─> Execute module.run()

4. Build Pipeline (modules/)
   ├─> PRE_BUILD: Set product, preloader
   ├─> PRE_LOAD: Preload configuration  
   ├─> LOAD: Load subsystems and parts
   ├─> GN: Generate ninja files
   └─> NINJA: Execute compilation

5. Completion
   └─> Log cost time, cleanup
```

### Independent Build Flow

```
hb build component_name -i

1. Detect -i flag in argument position
2. Validate indep_configs/build_indep.sh exists
3. Initialize IndepBuildModule with HPM + PrebuiltsService
4. Delegate to HPM for component resolution and build
```

### Argument Resolution Flow

```
1. Arg.parse_all_args(ModuleType) reads JSON definition
2. ArgsFactory creates argparse options
3. CLI arguments parsed
4. ArgsResolver maps args to resolve functions
5. Each phase triggers resolve_arg() for args in that phase
6. Resolvers configure services via regist_arg()
```

## Integration Points

### Upstream Dependencies
- **OS**: `os`, `sys`, `subprocess`, `platform`
- **Standard Library**: `json`, `shutil`, `glob`, `importlib`
- **Third-party**: `prompt_toolkit` (interactive menus), `kconfiglib`, `PyYAML`, `requests`

### Downstream Consumers (Dependencies created and injected)

| Layer | Components Injected |
|-------|-------------------|
| **Modules** | `OHOSBuildModule`, `OHOSSetModule`, `OHOSCleanModule`, etc. |
| **Resolvers** | `BuildArgsResolver`, `SetArgsResolver`, `CleanArgsResolver`, etc. |
| **Services** | `OHOSPreloader`, `OHOSLoader`, `Gn`, `Ninja`, `Hpm`, `Hdc` |

### External Tools Invoked
- **GN**: `prebuilts/build-tools/{platform}-{arch}/bin/gn`
- **Ninja**: `prebuilts/build-tools/{platform}-{arch}/bin/ninja`
- **HPM**: `prebuilts/hpm/node_modules/.bin/hpm` or system PATH
- **HDC**: Device debugging client
- **CCache**: Optional compiler caching

### Cross-Cutting Concerns
- **containers/**: Value objects (`Arg`, `Colors`), exception decorators (`throw_exception`)
- **exceptions/**: `OHOSException` with error code taxonomy
- **resources/**: Configuration singleton (`Config`), global variables, argument JSONs
- **helper/**: Metaclasses (`Singleton`, `NoInstance`), separators

## Module Directory Reference

| Directory | Purpose | See |
|-----------|---------|-----|
| `containers/` | Value objects, enums, decorators | [containers/codemap.md](containers/codemap.md) |
| `exceptions/` | Custom exception hierarchy | [exceptions/codemap.md](exceptions/codemap.md) |
| `helper/` | Metaclasses and utilities | [helper/codemap.md](helper/codemap.md) |
| `modules/` | Command execution implementations | [modules/codemap.md](modules/codemap.md) |
| `resolver/` | Argument parsing and resolution | [resolver/codemap.md](resolver/codemap.md) |
| `resources/` | Configuration and JSON definitions | [resources/codemap.md](resources/codemap.md) |
| `services/` | Core build service implementations | [services/codemap.md](services/codemap.md) |
| `util/` | Helper utilities and loaders | [util/codemap.md](util/codemap.md) |

## Key Design Decisions

1. **Workspace Validation**: Entry point validates OHOS source tree structure before loading to prevent runtime errors
2. **Dynamic Loading**: `__main__.py` uses `importlib` to load `main.py` from project directory, enabling context switching
3. **Independent Build Detection**: Dual-mode support (source vs component) via `judge_indep()` and `-i` flag detection
4. **Error Handling Strategy**: Centralized via `@throw_exception` decorator, externalized error metadata in JSON
5. **Configuration Persistence**: Arguments stored as JSON files, enabling cross-phase state sharing
6. **Phase-Based Execution**: Build process divided into phases (PRE_BUILD, PRE_LOAD, LOAD, etc.) for fine-grained control
7. **Service Injection**: Dependencies injected through constructors for testability and loose coupling
8. **Interactive Menu Support**: Uses `prompt_toolkit` for product selection UI in `hb set`

## File Count Summary

| Category | Count | Description |
|----------|-------|-------------|
| Python Files | ~85 | Main implementation |
| Interface Files | ~15 | Abstract base classes |
| JSON Configs | ~15 | Argument definitions, status codes |
| Test Files | ~3 | Unit tests |

## Entry Points

1. **hb command**: Defined in `setup.py` as `hb=hb.__main__:main`
2. **Module execution**: `python -m hb` triggers `hb/__main__.py`
3. **Direct execution**: `python main.py` (requires OHOS_ROOT context)
