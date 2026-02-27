# Containers Module Codemap

## 1. Responsibility

The `containers/` module serves as the **core domain model layer** for the HarmonyOS build system (hb). It defines fundamental data structures, enumerations, and cross-cutting concerns that underpin the entire build system's type system, argument processing, and error handling infrastructure.

### Key Functions:
- **Type System Definition**: Defines strongly-typed enumerations for build phases, module types, and argument types
- **Argument Encapsulation**: Provides the `Arg` class as the canonical representation of CLI arguments with lifecycle management
- **Exception Handling**: Implements centralized exception handling through the `throw_exception` decorator
- **Visual Output**: Supplies ANSI color constants for consistent terminal formatting

---

## 2. Design Patterns

### 2.1 Value Object / Data Transfer Object (DTO)
**File**: `arg.py` - Class: `Arg`

The `Arg` class implements the **Value Object** pattern (immutable data carrier) with controlled mutability for parsed values:

```python
class Arg:
    def __init__(self, name: str, helps: str, phase: str,
                 attribute: dict, argtype: ArgType, value,
                 resolve_function: str)
```

**Characteristics**:
- Encapsulates all metadata about a single command-line argument
- Property-based access control (`@property` decorators) with setter for `arg_value`
- Semantic equality based on `arg_name` (implied by `__str__` returning `name=value`)
- Self-contained serialization/deserialization logic

### 2.2 Static Factory Pattern
**File**: `arg.py` - Method: `Arg.create_instance_by_dict()`

Factory method for constructing `Arg` instances from JSON configuration:
- Parses JSON schema from `resources/args/default/*.json`
- Performs type coercion based on `ArgType` mapping
- Handles hyphen-to-underscore conversion for argument names
- Validates against unknown types with `OHOSException`

### 2.3 Enumeration Pattern
**File**: `arg.py`

Three enum-like classes provide type-safe constants:

| Class | Pattern | Purpose |
|-------|---------|---------|
| `ModuleType(Enum)` | Standard Python Enum | 11 build command categories (BUILD, SET, ENV, CLEAN, etc.) |
| `ArgType` | Class with static constants | Argument type system (BOOL, INT, STR, LIST, DICT, SUBPARSERS) |
| `BuildPhase` | Class with static constants | 13-phase build lifecycle |
| `CleanPhase` | Class with static constants | Clean operation modes (REGULAR, DEEP, NONE) |

**Type Resolution Strategy**:
Each non-Enum class provides `get_type(value: str)` static method for string-to-constant mapping, enabling deserialization from JSON configuration files.

### 2.4 Decorator Pattern
**File**: `status.py` - Function: `throw_exception`

Implements **Aspect-Oriented Programming (AOP)** for cross-cutting error handling:

```python
def throw_exception(func):
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        except OHOSException and Exception as exception:
            # Centralized error formatting and logging
            _print_formatted_tracebak(...)
            exit(-1)
    return wrapper
```

**Aspects handled**:
- Exception interception and classification
- Formatted error output to console
- Structured logging to `out/build.log`
- Process termination with error code

### 2.5 Dataclass Value Object
**File**: `colors.py` - Class: `Colors`

Simple **Value Object** using Python `@dataclass` decorator:
- Contains only class-level constants (ANSI escape codes)
- No instance state; used as namespace for color constants
- Used by `LogUtil` for consistent terminal styling

---

## 3. Data & Control Flow

### 3.1 Argument Initialization Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 1: Argument Definition (Static)                               │
│  resources/args/default/{module}args.json                           │
│  → JSON schema defining arg_name, arg_type, argDefault, etc.        │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 2: Deserialization                                          │
│  Arg.read_args_file(module_type)                                   │
│  → Checks CURRENT_ARGS_DIR existence                               │
│  → Copies default JSON if current args don't exist                 │
│  → Returns parsed dict via IoUtil.read_json_file()                 │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 3: Object Instantiation                                     │
│  Arg.create_instance_by_dict(json_entry)                           │
│  → Type coercion via ArgType.get_type()                            │
│  → Phase mapping via BuildPhase.get_type()                         │
│  → Default value casting based on arg_type                         │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 4: CLI Parsing                                              │
│  Arg.parse_all_args(module_type)                                   │
│  → ArgsFactory.genetic_add_option() builds argparse                │
│  → parser.parse_known_args() extracts user values                  │
│  → TypeCheckUtil.tile_list() flattens LIST/SUBPARSERS              │
│  → Arg.write_args_file() persists to CURRENT_ARGS_DIR              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Exception Handling Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Decorated Function Execution                                       │
│  @throw_exception                                                  │
│  def risky_operation():                                            │
│      raise OHOSException("Error", "0001")                          │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼ (exception raised)
┌─────────────────────────────────────────────────────────────────────┐
│  throw_exception.wrapper()                                         │
│  → Catches OHOSException or generic Exception                      │
│  → judge_indep() checks for independent build mode                 │
│  → If indep: raw exception + traceback.print_exc()                 │
│  → Else: formatted error via _print_formatted_tracebak()           │
└─────────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  _print_formatted_tracebak()                                       │
│  → Resolves log path from ROOT_CONFIG_FILE or default              │
│  → Writes full traceback to build.log                              │
│  → Writes structured error report (Code, Reason, Type, etc.)       │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Color Usage Flow

```
LogUtil.hb_warning() / LogUtil.write_log()
        │
        ▼
Colors.WARNING / Colors.ERROR / Colors.INFO
        │
        ▼
ANSI escape codes prepended to output strings
        │
        ▼
Terminal rendering with color formatting
```

---

## 4. Integration Points

### 4.1 Upstream Dependencies (Consumed by containers)

| Dependency | Usage Location | Purpose |
|------------|----------------|---------|
| `resources.global_var` | `arg.py` | File path constants for args JSON files |
| `exceptions.ohos_exception.OHOSException` | `arg.py`, `status.py` | Domain-specific exception type |
| `util.log_util.LogUtil` | `arg.py`, `status.py` | Structured logging to build.log |
| `util.io_util.IoUtil` | `arg.py`, `status.py` | JSON file I/O operations |
| `util.type_check_util.TypeCheckUtil` | `arg.py` | List flattening and type validation |
| `resolver.args_factory.ArgsFactory` | `arg.py` | argparse option generation |

### 4.2 Downstream Consumers (Users of containers)

#### Core Module Consumers:

| Consumer File | Imported Symbols | Usage Context |
|---------------|------------------|---------------|
| `main.py` | `Arg`, `ModuleType`, `throw_exception`, `judge_indep` | Entry point argument routing |
| `modules/ohos_build_module.py` | `BuildPhase`, `throw_exception` | Build lifecycle orchestration |
| `modules/ohos_clean_module.py` | `CleanPhase` | Clean operation mode selection |
| `modules/ohos_indep_build_module.py` | `BuildPhase` | Independent build phase management |
| `modules/interface/tool_module_interface.py` | `ModuleType`, `Arg` | Tool module contract |

#### Resolver Consumers (Argument Resolution):

| Consumer File | Imported Symbols | Purpose |
|---------------|------------------|---------|
| `resolver/build_args_resolver.py` | `Arg`, `throw_exception` | Build argument resolution |
| `resolver/clean_args_resolver.py` | `Arg` | Clean argument resolution |
| `resolver/env_args_resolver.py` | `Arg`, `ModuleType` | Environment argument resolution |
| `resolver/set_args_resolver.py` | `Arg`, `ModuleType` | Set argument resolution |
| `resolver/indep_build_args_resolver.py` | `Arg`, `ModuleType` | Independent build resolution |
| `resolver/install_args_resolver.py` | `Arg`, `ModuleType` | Install command resolution |
| `resolver/package_args_resolver.py` | `Arg`, `ModuleType` | Package command resolution |
| `resolver/publish_args_resolver.py` | `Arg`, `ModuleType` | Publish command resolution |
| `resolver/push_args_resolver.py` | `Arg`, `ModuleType` | Push command resolution |
| `resolver/update_args_resolver.py` | `Arg`, `ModuleType` | Update command resolution |
| `resolver/tool_args_resolver.py` | `Arg`, `throw_exception` | Tool command resolution |
| `resolver/interface/args_resolver_interface.py` | `Arg`, `throw_exception` | Resolver base interface |

#### Service Layer Consumers:

| Consumer File | Imported Symbols | Purpose |
|---------------|------------------|---------|
| `services/gn.py` | `Arg`, `ModuleType`, `throw_exception` | GN build system integration |
| `services/loader.py` | `throw_exception` | Build configuration loading |
| `services/menu.py` | `Arg`, `ModuleType` | Interactive menu system |
| `services/hpm.py` | `throw_exception` | HPM package manager integration |
| `services/hdc.py` | `throw_exception`, `Arg`, `ModuleType` | HDC device communication |
| `services/prebuilt_sdk.py` | `Arg` | Prebuilt SDK management |

#### Utility Consumers:

| Consumer File | Imported Symbols | Purpose |
|---------------|------------------|---------|
| `util/log_util.py` | `Colors` | Terminal color formatting |
| `util/product_util.py` | `throw_exception` | Product configuration utilities |
| `util/system_util.py` | `throw_exception` | System-level operations |
| `util/device_util.py` | `throw_exception` | Device management utilities |
| `util/component_util.py` | `throw_exception` | Component utilities |
| `util/loader/*.py` | `throw_exception` | Various loader utilities |

### 4.3 File System Integration

#### Configuration File Layout:

```
resources/args/default/
├── buildargs.json      # Default build arguments (ModuleType.BUILD)
├── setargs.json        # Default set arguments (ModuleType.SET)
├── cleanargs.json      # Default clean arguments (ModuleType.CLEAN)
├── envargs.json        # Default env arguments (ModuleType.ENV)
├── toolargs.json       # Default tool arguments (ModuleType.TOOL)
├── indepbuildargs.json # Default independent build args (ModuleType.INDEP_BUILD)
├── installargs.json    # Default install arguments (ModuleType.INSTALL)
├── packageargs.json    # Default package arguments (ModuleType.PACKAGE)
├── publishargs.json    # Default publish arguments (ModuleType.PUBLISH)
├── updateargs.json     # Default update arguments (ModuleType.UPDATE)
└── pushargs.json       # Default push arguments (ModuleType.PUSH)

out/hb_args/            # Runtime argument state directory
├── buildargs.json      # Current build arguments (copied from default)
├── setargs.json        # Current set arguments
└── ...                 # Other module current args
```

#### State Persistence Model:
- **Defaults**: Immutable configuration in `resources/args/default/`
- **Current State**: Mutable runtime state in `out/hb_args/`
- **Lifecycle**: Args copied from default → parsed → modified → persisted back

---

## 5. Module Type Matrix

| ModuleType | Default Args File | Current Args File | Primary Resolver | Module Class |
|------------|-------------------|-------------------|------------------|--------------|
| BUILD | buildargs.json | buildargs.json | build_args_resolver.py | ohos_build_module.py |
| SET | setargs.json | setargs.json | set_args_resolver.py | ohos_set_module.py |
| ENV | envargs.json | envargs.json | env_args_resolver.py | ohos_env_module.py |
| CLEAN | cleanargs.json | cleanargs.json | clean_args_resolver.py | ohos_clean_module.py |
| TOOL | toolargs.json | toolargs.json | tool_args_resolver.py | ohos_tool_module.py |
| INDEP_BUILD | indepbuildargs.json | indepbuildargs.json | indep_build_args_resolver.py | ohos_indep_build_module.py |
| INSTALL | installargs.json | installargs.json | install_args_resolver.py | ohos_install_module.py |
| PACKAGE | packageargs.json | packageargs.json | package_args_resolver.py | ohos_package_module.py |
| PUBLISH | publishargs.json | publishargs.json | publish_args_resolver.py | ohos_publish_module.py |
| UPDATE | updateargs.json | updateargs.json | update_args_resolver.py | ohos_update_module.py |
| PUSH | pushargs.json | pushargs.json | push_args_resolver.py | ohos_push_module.py |

---

## 6. Build Phase Lifecycle

The `BuildPhase` class defines a 13-phase build pipeline:

```
PRE_BUILD (1)
    ↓
PRE_LOAD (2) → HPM_DOWNLOAD (11) [alternative path]
    ↓
LOAD (3)
    ↓
PRE_TARGET_GENERATE (4)
    ↓
TARGET_GENERATE (5)
    ↓
POST_TARGET_GENERATE (6)
    ↓
PRE_TARGET_COMPILATION (7)
    ↓
TARGET_COMPILATION (8)
    ↓
POST_TARGET_COMPILATION (9)
    ↓
POST_BUILD (10)
    ↓
INDEP_COMPILATION (12) [independent build only]
```

Each `Arg` instance can be associated with one or more phases via the `arg_phase` attribute, enabling phase-specific argument resolution.

---

## 7. Error Code Conventions

The `throw_exception` decorator uses error codes where the **first digit indicates the build stage**:

| First Digit | Stage | Description |
|-------------|-------|-------------|
| 0 | General | Initialization, argument parsing, generic errors |
| 1 | Preloader | Pre-build configuration loading |
| 2 | Loader | Build target loading |
| 3 | GN | GN meta-build system |
| 4 | Ninja | Ninja build execution |

Example: `OHOSException("...", "0003")` → Type error during argument parsing
Example: `OHOSException("...", "1001")` → Error in preloader stage

---

## 8. Summary

The `containers/` module is the **architectural foundation** of the hb build system:

1. **Type Safety**: Strongly-typed enumerations prevent invalid state transitions
2. **Configuration-as-Data**: JSON-driven argument definitions enable declarative CLI design
3. **Centralized Error Handling**: AOP-style exception decoration ensures consistent error UX
4. **Separation of Concerns**: Pure data containers (`Arg`, `Colors`) separated from behavior (`throw_exception`)

This module exhibits **Domain-Driven Design (DDD)** principles with clear bounded contexts for argument management, status tracking, and visual presentation.
