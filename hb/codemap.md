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

### 5. **Decorator Chain**
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

## Integration Points

### Upstream Dependencies
- **OS**: `os`, `sys`, `subprocess`, `platform`
- **Standard Library**: `json`, `shutil`, `glob`, `importlib`

### Downstream Consumers (Dependencies created and injected)

| Layer | Components Injected |
|-------|-------------------|
| **Modules** | `OHOSBuildModule`, `OHOSSetModule`, `OHOSCleanModule`, etc. |
| **Resolvers** | `BuildArgsResolver`, `SetArgsResolver`, `CleanArgsResolver`, etc. |
| **Services** | `OHOSPreloader`, `OHOSLoader`, `Gn`, `Ninja`, `Hpm`, `Hdc` |

### Cross-Cutting Concerns
- **containers/**: Value objects (`Arg`, `Colors`), exception decorators (`throw_exception`)
- **exceptions/**: `OHOSException` with error code taxonomy
- **resources/**: Configuration singleton (`Config`), global variables, argument JSONs
- **helper/**: Metaclasses (`Singleton`, `NoInstance`), separators

### Module Directory Reference

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
