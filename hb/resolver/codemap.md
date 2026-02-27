# Resolver Module Codemap

## Overview

The `resolver/` directory implements the **Argument Resolution Subsystem** for the HarmonyOS hb (build) command-line tool. This subsystem is responsible for parsing, validating, and transforming command-line arguments into actionable build configurations.

---

## 1. Responsibility

### 1.1 Core Purpose

The resolver module serves as the **intermediary layer** between raw CLI input and the build system's internal representation. Its primary responsibilities include:

- **Argument Parsing**: Transforming CLI strings into typed `Arg` objects
- **Validation**: Ensuring argument values meet semantic constraints
- **Resolution**: Converting arguments into build system configurations
- **Persistence**: Managing argument state across build invocations
- **Phase Coordination**: Executing resolution logic at appropriate build phases

### 1.2 Resolution Phases

The build system defines 12 distinct phases (`BuildPhase` enum in `containers/arg.py`):

| Phase | Enum Value | Description |
|-------|------------|-------------|
| PRE_BUILD | 1 | Initial setup, environment validation |
| PRE_LOAD | 2 | Pre-loading configuration |
| LOAD | 3 | Loading subsystem configurations |
| PRE_TARGET_GENERATE | 4 | Before GN target generation |
| TARGET_GENERATE | 5 | GN target generation phase |
| POST_TARGET_GENERATE | 6 | After GN target generation |
| PRE_TARGET_COMPILATION | 7 | Before ninja compilation |
| TARGET_COMPILATION | 8 | Ninja compilation phase |
| POST_TARGET_COMPILATION | 9 | Post-compilation processing |
| POST_BUILD | 10 | Final build steps |
| HPM_DOWNLOAD | 11 | HPM package download phase |
| INDEP_COMPILATION | 12 | Independent component compilation |

---

## 2. Architecture & Design Patterns

### 2.1 Strategy Pattern

Each command type (build, clean, env, etc.) has a dedicated **ArgsResolver** class that implements command-specific resolution strategies.

```
ArgsResolverInterface (Abstract Base)
    ├── BuildArgsResolver       → Build command resolution
    ├── CleanArgsResolver       → Clean command resolution
    ├── EnvArgsResolver         → Environment command resolution
    ├── SetArgsResolver         → Set command resolution
    ├── ToolArgsResolver        → Tool command resolution
    ├── IndepBuildArgsResolver  → Independent build resolution
    ├── InstallArgsResolver     → Install command resolution
    ├── PackageArgsResolver     → Package command resolution
    ├── PublishArgsResolver     → Publish command resolution
    ├── UpdateArgsResolver      → Update command resolution
    ├── PushArgsResolver        → Push command resolution
    └── JudgeIndepArgsResolver  → Independent build eligibility
```

**Pattern Implementation**:
- Interface: `ArgsResolverInterface` defines the contract
- Concrete Strategies: Each `*ArgsResolver` implements resolution logic
- Context: The module classes delegate to their respective resolver

### 2.2 Factory Pattern

`ArgsFactory` provides a **factory method** for creating argparse options based on argument type metadata.

**Factory Method**: `genetic_add_option(parser, arg_dict)`

**Product Types**:
| Type | Factory Method | Description |
|------|---------------|-------------|
| bool | `_add_bool_option()` | Boolean flags (true/false) |
| str | `_add_str_option()` | String arguments |
| list | `_add_list_option()` | List arguments (nargs='*') |
| subparsers | `_add_list_option()` | Subcommand arguments |

**Configuration-Driven**: Argument definitions are stored in JSON files (`resources/args/`), enabling declarative argument specification without code changes.

### 2.3 Template Method Pattern

`ArgsResolverInterface` defines a template for argument resolution:

1. **Initialization**: `__init__(args_dict)` → maps args to resolution functions
2. **Mapping**: `_map_args_to_function()` → validates resolution function existence
3. **Resolution**: `resolve_arg()` → delegates to appropriate resolver function

### 2.4 Command Pattern

Each static resolution method acts as a **command** that:
- Receives: `Arg` object + Module interface
- Performs: Side effects on module components
- Returns: None (mutates state)

Example resolution method signature:
```python
@staticmethod
def resolve_product(target_arg: Arg, build_module: BuildModuleInterface):
    # Extract value from Arg
    # Configure module components
    # Register with target_generator/loader/compiler
```

---

## 3. Data & Control Flow

### 3.1 Argument Lifecycle

```
┌─────────────────────────────────────────────────────────────────────┐
│                      ARGUMENT LIFECYCLE                              │
└─────────────────────────────────────────────────────────────────────┘

    CLI Input
       │
       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  JSON Definition │───▶│   ArgsFactory    │───▶│  Arg Objects    │
│  (args/*.json)   │    │  (Factory)       │    │  (Typed)        │
└─────────────────┘    └──────────────────┘    └─────────────────┘
                                                          │
       ┌──────────────────────────────────────────────────┘
       ▼
┌─────────────────┐    ┌──────────────────┐    ┌─────────────────┐
│  Module         │◄───│  *ArgsResolver   │◄───│  Arg.parse_*   │
│  Components     │    │  (Strategy)      │    │  (argparse)     │
│  (Generator,    │    │                  │    │                 │
│   Compiler)     │    │  resolve_*()     │    │  Type coercion  │
└─────────────────┘    └──────────────────┘    └─────────────────┘
       │
       ▼
┌─────────────────┐
│  Build System   │
│  Execution      │
└─────────────────┘
```

### 3.2 Resolution Flow Detail

#### Step 1: Argument Definition (Static)
Arguments are defined in JSON files with metadata:
```json
{
  "arg_name": "--product-name",
  "arg_help": "Specify product name",
  "arg_type": "str",
  "argDefault": "",
  "arg_phase": "prebuild",
  "resolve_function": "resolve_product_name",
  "arg_attribute": {"abbreviation": "-p"}
}
```

#### Step 2: Parser Construction
`Arg.parse_all_args(ModuleType)`:
1. Reads JSON definition file
2. Invokes `ArgsFactory.genetic_add_option()` for each argument
3. Parses `sys.argv` via argparse
4. Creates `Arg` instances with typed values
5. Persists to JSON state files

#### Step 3: Resolution Mapping
`ArgsResolverInterface.__init__()`:
```python
for entity in args_dict.values():
    function_name = entity.resolve_function  # e.g., "resolve_product"
    entity.resolve_function = self.__getattribute__(function_name)
    self._args_to_function[entity.arg_name] = entity.resolve_function
```

#### Step 4: Phase-Based Execution
The module orchestrates resolution by phase:
```python
# In module (e.g., ohos_build_module.py)
for phase in BuildPhase:
    for arg in args_dict.values():
        if arg.arg_phase == phase:
            resolver.resolve_arg(arg, self)
```

### 3.3 State Persistence Flow

Arguments are persisted across invocations using JSON files:

| Module Type | Current Args File | Default Args File |
|-------------|-------------------|-------------------|
| BUILD | `CURRENT_BUILD_ARGS` | `DEFAULT_BUILD_ARGS` |
| SET | `CURRENT_SET_ARGS` | `DEFAULT_SET_ARGS` |
| ENV | `CURRENT_ENV_ARGS` | `DEFAULT_ENV_ARGS` |
| CLEAN | `CURRENT_CLEAN_ARGS` | `DEFAULT_CLEAN_ARGS` |
| TOOL | `CURRENT_TOOL_ARGS` | `DEFAULT_TOOL_ARGS` |
| INDEP_BUILD | `CURRENT_INDEP_BUILD_ARGS` | `DEFAULT_INDEP_BUILD_ARGS` |

**Storage Location**: `~/.hb/args/` (via `CURRENT_ARGS_DIR`)

**Methods**:
- `Arg.write_args_file(key, value, module_type)` → Persists single argument
- `Arg.read_args_file(module_type)` → Loads all arguments
- `Arg.clean_args_file()` → Removes all persisted arguments

---

## 4. File Structure & Component Analysis

### 4.1 Core Files

#### `interface/args_resolver_interface.py`
**Purpose**: Abstract base class for all argument resolvers

**Key Methods**:
- `__init__(args_dict)`: Initializes the resolver, maps args to functions
- `resolve_arg(target_arg, module)`: Executes resolution for a single argument
- `_map_args_to_function(args_dict)`: Validates and binds resolution functions

**Error Handling**: Uses `@throw_exception` decorator for consistent exception handling

---

#### `args_factory.py`
**Purpose**: Factory for creating argparse.ArgumentParser options

**Class**: `ArgsFactory`
**Primary Method**: `genetic_add_option(parser, arg_dict)`

**Helper Functions**:
- `_add_bool_option()` / `_add_bool_abbreviation_option()`
- `_add_str_option()` / `_add_str_abbreviation_option()` / `_add_str_optional_option()`
- `_add_list_option()` / `_add_list_abbreviation_option()`

---

#### `build_args_resolver.py`
**Purpose**: Resolution logic for `hb build` command

**Class**: `BuildArgsResolver`
**Lines**: 1057

**Key Resolution Methods**:

| Method | Phase | Description |
|--------|-------|-------------|
| `resolve_product()` | prebuild | Configures product, device, kernel paths |
| `resolve_build_target()` | prebuild | Transforms targets (TDD, precise, component) |
| `resolve_target_cpu()` | prebuild | Sets target CPU architecture |
| `resolve_ccache()` | prebuild | Configures ccache environment |
| `resolve_xcache()` | prebuild | Configures xcache distributed cache |
| `resolve_gn_args()` | prebuild | Parses key=value GN arguments |
| `resolve_gn_flags()` | targetGenerate | Passes raw GN flags |
| `resolve_ninja_args()` | prebuild | Passes ninja build arguments |
| `resolve_strict_mode()` | load | Validates preloader/loader outputs |
| `resolve_test()` | targetGenerate | Configures test compilation |
| `resolve_build_type()` | targetGenerate | Sets debug/profile build type |
| `resolve_device_type()` | postTargetCompilation | Modifies ohos.para characteristics |
| `resolve_archive_image()` | postTargetCompilation | Creates tar.gz of images |
| `resolve_patch()` | postTargetCompilation | Applies post-build patches |
| `resolve_rom_size_statistics()` | postTargetCompilation | Generates ROM statistics |
| `resolve_deps_guard()` | postbuild | Validates dependency compliance |

**Special Features**:
- TDD (Test-Driven Development) target resolution from CSV manifests
- Precise compilation support for incremental builds
- ccache/xcache integration with environment variable management
- Component directory detection for partial builds

---

#### `clean_args_resolver.py`
**Purpose**: Resolution logic for `hb clean` command

**Class**: `CleanArgsResolver`

**Resolution Methods**:
- `resolve_clean_args()`: Cleans persisted argument files
- `resolve_clean_out_product()`: Removes output directory
- `resolve_clean_ccache()`: Clears ccache directory
- `resolve_clean_all()`: Orchestrates full cleanup

---

#### `env_args_resolver.py`
**Purpose**: Resolution logic for `hb env` command

**Class**: `EnvArgsResolver`

**Resolution Methods**:
- `resolve_check()`: Validates environment, displays package status
- `resolve_install()`: Executes dependency installation script
- `resolve_clean()`: Cleans ENV module arguments

---

#### `set_args_resolver.py`
**Purpose**: Resolution logic for `hb set` command

**Class**: `SetArgsResolver`

**Resolution Methods**:
- `resolve_product_name()`: Configures product, derives device/board info
- `resolve_set_parameter()`: Interactive menu for compilation options

**Key Behavior**: Derives `Config` singleton properties from product selection, including:
- Product path, OS level, version
- Board, kernel, target CPU/OS
- Output path calculation
- Subsystem configuration paths

---

#### `tool_args_resolver.py`
**Purpose**: Resolution logic for `hb tool` command

**Class**: `ToolArgsResolver`

**Resolution Methods**:
- `resolve_list_targets()`: Lists GN targets
- `resolve_desc_targets()`: Describes target properties
- `resolve_path_targets()`: Shows target dependency paths
- `resolve_refs_targets()`: Shows target references
- `resolve_format_targets()`: Formats GN files
- `resolve_clean_targets()`: Cleans GN output

**Pattern**: All methods delegate to `GN` service with `CMDTYPE` enum

---

#### `indep_build_args_resolver.py`
**Purpose**: Resolution logic for `hb build -i` (independent build)

**Class**: `IndepBuildArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Resolves component bundle.json paths (with ccache)
- `resolve_target_cpu()` / `resolve_target_os()`: Cross-compilation settings
- `resolve_variant()`: Product variant selection
- `resolve_branch()`: Source branch configuration
- `resolve_build_type()`: Source-only/test/both build types
- `resolve_ccache()`: ccache setup for independent builds
- `resolve_gn_args()` / `resolve_gn_flags()` / `resolve_ninja_args()`: Build configuration

**Key Feature**: Component bundle.json path caching (`COMPONENTS_PATH_DIR`)

---

#### `judge_indep_args_resolver.py`
**Purpose**: Determines if build qualifies for independent compilation

**Class**: `ArgsResolver`

**Static Methods**:
- `is_indep_args(input_args)`: Main entry point, returns (bool, components)
- `parse_command_args()`: Converts sys.argv to dict
- `read_indep_whitelist()`: Loads whitelist from JSON
- `parse_component_name()`: Extracts component from build target
- `find_deepest_bundle_json()`: Locates bundle.json in target path

**Eligibility Criteria**:
1. Product is in whitelist
2. All build targets are whitelisted components
3. No unsupported arguments present

---

#### `install_args_resolver.py`
**Purpose**: Resolution logic for `hb install` command

**Class**: `InstallArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Component name for installation
- `resolve_global()`: Global installation flag
- `resolve_local()`: Local installation path
- `resolve_variant()`: Dependency variant

**Integration**: Registers flags with HPM (HarmonyOS Package Manager) service

---

#### `package_args_resolver.py`
**Purpose**: Resolution logic for `hb package` command

**Class**: `PackageArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Component to package
- `resolve_output()`: Output directory

---

#### `publish_args_resolver.py`
**Purpose**: Resolution logic for `hb publish` command

**Class**: `PublishArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Component to publish

---

#### `update_args_resolver.py`
**Purpose**: Resolution logic for `hb update` command

**Class**: `UpdateArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Component to update
- `resolve_global()`: Global update flag

---

#### `push_args_resolver.py`
**Purpose**: Resolution logic for `hb push` command

**Class**: `PushArgsResolver`

**Resolution Methods**:
- `resolve_part()`: Component to push
- `resolve_target()`: Target device
- `resolve_list_targets()`: Lists connected devices
- `resolve_reboot()`: Reboot after push
- `resolve_src()`: Source path override

**Integration**: Registers flags with HDC (HarmonyOS Device Connector) service

---

## 5. Integration Points

### 5.1 Module Consumers

Each resolver integrates with a corresponding module interface:

```
Resolver                      Module Interface                    Module Implementation
─────────────────────────────────────────────────────────────────────────────────────────
BuildArgsResolver      →      BuildModuleInterface        →      ohos_build_module.py
CleanArgsResolver      →      CleanModuleInterface        →      ohos_clean_module.py
EnvArgsResolver        →      EnvModuleInterface          →      ohos_env_module.py
SetArgsResolver        →      SetModuleInterface          →      ohos_set_module.py
ToolArgsResolver       →      ToolModuleInterface         →      ohos_tool_module.py
IndepBuildArgsResolver →      IndepBuildModuleInterface   →      ohos_indep_build_module.py
InstallArgsResolver    →      InstallModuleInterface      →      ohos_install_module.py
PackageArgsResolver    →      PackageModuleInterface      →      ohos_package_module.py
PublishArgsResolver    →      PublishModuleInterface      →      ohos_publish_module.py
UpdateArgsResolver     →      UpdateModuleInterface       →      ohos_update_module.py
PushArgsResolver       →      PushModuleInterface         →      ohos_push_module.py
```

### 5.2 Service Integration

Resolvers register arguments with service executors:

| Service | Registration Method | Purpose |
|---------|-------------------|---------|
| `target_generator` | `regist_arg(name, value)` | GN build arguments |
| `target_generator` | `regist_flag(name, value)` | GN command flags |
| `target_compiler` | `regist_arg(name, value)` | Ninja arguments |
| `loader` | `regist_arg(name, value)` | Subsystem loading args |
| `preloader` | `regist_arg(name, value)` | Preloading configuration |
| `hpm` | `regist_flag(name, value)` | HPM package manager |
| `indep_build` | `regist_flag(name, value)` | Independent build executor |
| `hdc` | `regist_flag(name, value)` | Device connector |
| `gn` | `execute_gn_cmd()` | GN tool commands |

### 5.3 Configuration Integration

The `Config` singleton (`resources/config.py`) is accessed by multiple resolvers:
- `BuildArgsResolver`: Reads/updates product, board, kernel settings
- `SetArgsResolver`: Sets all configuration properties
- `EnvArgsResolver`: Displays configuration status

### 5.4 Utility Integration

| Utility | Usage |
|---------|-------|
| `ComponentUtil` | Bundle.json search, component validation |
| `ProductUtil` | Product info, device info, features extraction |
| `DeviceUtil` | Device path resolution |
| `IoUtil` | JSON file I/O |
| `LogUtil` | User-facing messages |
| `SystemUtil` | Command execution |
| `TypeCheckUtil` | Type coercion and validation |

---

## 6. Key Design Decisions

### 6.1 Static Method Resolution Functions
All resolution methods are `@staticmethod`, enabling:
- Statelessness (no instance state needed)
- Easy testing (no module mocking required)
- Functional composition

### 6.2 Decorator-Based Error Handling
The `@throw_exception` decorator provides:
- Consistent error codes
- Stack trace logging
- Graceful degradation

### 6.3 JSON-Driven Argument Definitions
Arguments are defined externally to enable:
- Non-code modifications to CLI
- Phase assignment without code changes
- Help text internationalization potential

### 6.4 Module Type Enumeration
`ModuleType` enum provides type-safe argument scoping:
- Prevents cross-module argument pollution
- Enables module-specific persistence
- Supports command validation

---

## 7. Extension Guidelines

### Adding a New Argument

1. **Define in JSON**: Add entry to appropriate `resources/args/*.json`
2. **Implement Resolver**: Add `resolve_*` method to corresponding `*ArgsResolver`
3. **Register in Module**: Ensure module calls `resolver.resolve_arg()` at appropriate phase

### Adding a New Command

1. **Create Interface**: Define `*ModuleInterface` in `modules/interface/`
2. **Implement Module**: Create `ohos_*_module.py` in `modules/`
3. **Create Resolver**: Extend `ArgsResolverInterface` in `resolver/*_args_resolver.py`
4. **Add ModuleType**: Extend `ModuleType` enum in `containers/arg.py`
5. **Define Arguments**: Create `resources/args/default_*_args.json`

---

## 8. Error Codes

The resolver subsystem uses the following error codes (from `OHOSException`):

| Code | Meaning | Trigger |
|------|---------|---------|
| 0000 | Missing file/resource | Env setup file not found |
| 0001 | Invalid format | Malformed GN args (expected key=value) |
| 0002 | Invalid value | Unsupported test type, unknown module |
| 0003 | Unknown type | Unrecognized arg_type in JSON |
| 0004 | Missing function | Resolution function not implemented |
| 0018 | Unknown module type | Invalid ModuleType enum value |
| 1001 | Preloader failure | Strict mode preloader validation |
| 2001 | Loader failure | Strict mode loader validation |
| 4001 | Component not found | Build target doesn't exist in product |

---

*Document generated for HarmonyOS hb build system - Resolver Module*
