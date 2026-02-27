# CodeMap: modules/interface/ Directory

## Overview

The `modules/interface/` directory defines the **Abstract Base Class (ABC) interface layer** for the HarmonyOS build system (hb). This directory implements the **Command Pattern** and **Template Method Pattern** to standardize how different build operations are executed.

**Location**: `/home/zhuojw/a_git/ohos-mani-v2/build/hb/modules/interface/`

---

## 1. Responsibility

This directory provides **interface definitions for all hb command modules**, establishing a contractual boundary between:
- **CLI argument parsing** (`resolver/` layer)
- **Service implementations** (`services/` layer)
- **Command orchestration** (`modules/` layer)

Each interface defines:
- The lifecycle methods for a specific command type
- Required service dependencies
- Execution workflow via the `run()` template method

---

## 2. File Inventory

| File | Interface Class | Purpose |
|------|----------------|---------|
| `module_interface.py` | `ModuleInterface` | Root ABC for all modules |
| `build_module_interface.py` | `BuildModuleInterface` | Full build workflow (preload → load → gn → ninja) |
| `clean_module_interface.py` | `CleanModuleInterface` | Clean operations (regular + deep) |
| `env_module_interface.py` | `EnvModuleInterface` | Environment setup workflow |
| `indep_build_module_interface.py` | `IndepBuildModuleInterface` | Independent/targeted build |
| `install_module_interface.py` | `InstallModuleInterface` | Install operations |
| `package_module_interface.py` | `PackageModuleInterface` | Package creation |
| `publish_module_interface.py` | `PublishModuleInterface` | Publish operations |
| `push_module_interface.py` | `PushModuleInterface` | Push to device |
| `set_module_interface.py` | `SetModuleInterface` | Configuration management |
| `tool_module_interface.py` | `ToolModuleInterface` | Development tools (ls, desc, path, refs, format, clean) |
| `update_module_interface.py` | `UpdateModuleInterface` | Update operations |

---

## 3. Design Patterns

### 3.1 Abstract Base Class (ABC) Pattern

All interfaces use Python's `abc.ABCMeta` metaclass with `@abstractmethod` decorators:

```python
class ModuleInterface(metaclass=ABCMeta):
    @abstractmethod
    def run(self):
        pass  # Force implementation in concrete classes
```

**Benefits**:
- Enforces method implementation at runtime
- Prevents instantiation of incomplete classes
- Establishes polymorphic contract

### 3.2 Template Method Pattern

Each interface's `run()` method defines the **algorithm skeleton**, delegating specific steps to subclasses:

**Example - BuildModuleInterface.run()** (lines 63-74):
```python
def run(self):
    try:
        self._prebuild_and_preload()  # Phase 1
        self._load()                   # Phase 2
        self._gn()                     # Phase 3
        self._ninja()                  # Phase 4
    except OHOSException as exception:
        raise exception
    else:
        self._post_target_compilation()
    finally:
        self._post_build()
```

**Example - CleanModuleInterface.run()** (lines 37-39):
```python
def run(self):
    self.clean_regular()
    self.clean_deep()
```

### 3.3 Interface Segregation Principle (ISP)

Each interface is **highly cohesive** and focused on a single responsibility:

- Single-operation interfaces: `Install`, `Package`, `Publish`, `Push`, `Update`
- Multi-phase interfaces: `Build`, `Clean`, `Env`, `Set`
- Multi-tool interface: `Tool` (6 distinct operations)

This prevents "fat interface" anti-pattern and ensures implementers only depend on methods they need.

### 3.4 Dependency Injection (DI)

Interfaces receive their dependencies via constructor injection:

**Base ModuleInterface** (lines 25-27):
```python
def __init__(self, args_dict: dict, args_resolver: ArgsResolverInterface):
    self._args_dict = args_dict
    self._args_resolver = args_resolver
```

**BuildModuleInterface** (lines 33-45) - Extended DI:
```python
def __init__(self, args_dict: dict,
             args_resolver: ArgsResolverInterface,
             preloader: PreloadInterface,          # Service layer
             loader: LoadInterface,                # Service layer
             target_generator: BuildFileGeneratorInterface,  # Service layer
             target_compiler: BuildExecutorInterface)        # Service layer
```

### 3.5 Decorator Pattern (Timing)

`BuildModuleInterface` uses `@TimerUtil.cost_time` decorator on expensive operations:

```python
@TimerUtil.cost_time
@abstractmethod
def _prebuild_and_preload(self):
    self._prebuild()
    self._preload()
```

---

## 4. Class Hierarchy

```
ModuleInterface (ABC)
├── BuildModuleInterface
├── CleanModuleInterface
├── EnvModuleInterface
├── IndepBuildModuleInterface
├── InstallModuleInterface
├── PackageModuleInterface
├── PublishModuleInterface
├── PushModuleInterface
├── SetModuleInterface
├── ToolModuleInterface
└── UpdateModuleInterface
```

### 4.1 ModuleInterface - Root Abstract Base Class

**Location**: `module_interface.py`

**Dependencies**:
- `resolver.interface.args_resolver_interface.ArgsResolverInterface`

**Contract**:
| Property | Type | Description |
|----------|------|-------------|
| `args_dict` | `dict` | Parsed CLI arguments mapping |
| `args_resolver` | `ArgsResolverInterface` | Argument resolver instance |

**Abstract Method**:
- `run()` - Main execution entry point (must be implemented by all subclasses)

---

### 4.2 BuildModuleInterface - Full Build Workflow

**Location**: `build_module_interface.py`

**Extends**: `ModuleInterface`

**Purpose**: Orchestrates the complete HarmonyOS build pipeline with 4 distinct phases.

**Service Dependencies** (injected):
| Dependency | Interface | Phase Used |
|------------|-----------|------------|
| `preloader` | `PreloadInterface` | Phase 1 |
| `loader` | `LoadInterface` | Phase 2 |
| `target_generator` | `BuildFileGeneratorInterface` | Phase 3 |
| `target_compiler` | `BuildExecutorInterface` | Phase 4 |

**4-Phase Execution Flow**:

```
┌─────────────────────────────────────────────────────────────────┐
│  Phase 1: PREBUILD & PRELOAD (@TimerUtil.cost_time)            │
│  ├── _prebuild()     [abstract] - Environment preparation       │
│  └── _preload()      [abstract] - Load build configuration      │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Phase 2: LOAD                                                  │
│  └── _load()         [abstract] - Parse build files             │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Phase 3: GN - Generate Ninja Files (@TimerUtil.cost_time)     │
│  ├── _pre_target_generate()   [abstract] - Pre-generation hook  │
│  ├── _target_generate()       [abstract] - GN execution         │
│  └── _post_target_generate()  [abstract] - Post-generation hook │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Phase 4: NINJA - Compilation (@TimerUtil.cost_time)           │
│  ├── _pre_target_compilation()  [abstract] - Pre-build hook     │
│  └── _target_compilation()      [abstract] - Ninja execution    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Post-Execution (else block)                                    │
│  └── _post_target_compilation() [abstract] - Success handler    │
└─────────────────────────────────────────────────────────────────┘
                              ↓
┌─────────────────────────────────────────────────────────────────┐
│  Cleanup (finally block)                                        │
│  └── _post_build()    [abstract] - Always executes              │
└─────────────────────────────────────────────────────────────────┘
```

**Error Handling**: Uses `OHOSException` for error propagation with `try/except/else/finally` structure.

---

### 4.3 CleanModuleInterface - Artifact Cleanup

**Location**: `clean_module_interface.py`

**Extends**: `ModuleInterface`

**2-Phase Workflow**:
1. `clean_regular()` - Standard cleanup (incremental build artifacts)
2. `clean_deep()` - Deep cleanup (all generated files, full reset)

---

### 4.4 EnvModuleInterface - Environment Management

**Location**: `env_module_interface.py`

**Extends**: `ModuleInterface`

**3-Phase Workflow**:
1. `env_check()` - Validate system dependencies
2. `env_install()` - Install missing dependencies
3. `clean()` - Cleanup temporary files

---

### 4.5 IndepBuildModuleInterface - Targeted Build

**Location**: `indep_build_module_interface.py`

**Extends**: `ModuleInterface`

**Purpose**: Independent/targeted compilation for specific build targets without full workflow.

**Single Operation**:
- `_target_compilation()` - Compile specified targets only

**Error Handling**: Wraps execution in `try/except OHOSException`.

---

### 4.6 InstallModuleInterface - Installation

**Location**: `install_module_interface.py`

**Extends**: `ModuleInterface`

**Single Operation**:
- `_install()` - Install build artifacts

---

### 4.7 PackageModuleInterface - Packaging

**Location**: `package_module_interface.py`

**Extends**: `ModuleInterface`

**Single Operation**:
- `_package()` - Create distributable packages

---

### 4.8 PublishModuleInterface - Publishing

**Location**: `publish_module_interface.py`

**Extends**: `ModuleInterface`

**Single Operation**:
- `_publish()` - Publish artifacts to repositories

---

### 4.9 PushModuleInterface - Device Deployment

**Location**: `push_module_interface.py`

**Extends**: `ModuleInterface`

**Single Operation**:
- `_push()` - Push artifacts to connected devices

---

### 4.10 SetModuleInterface - Configuration

**Location**: `set_module_interface.py`

**Extends**: `ModuleInterface`

**Conditional 2-Phase Workflow** (lines 37-40):
```python
def run(self):
    if not self.args_dict['all'].arg_value:
        self.set_product()      # Set product configuration
    self.set_parameter()        # Set build parameters
```

**Operations**:
- `set_product()` - Configure product settings (skipped if `--all` flag set)
- `set_parameter()` - Configure build parameters (always executed)

---

### 4.11 ToolModuleInterface - Development Tools

**Location**: `tool_module_interface.py`

**Extends**: `ModuleInterface`

**Purpose**: Multi-tool interface providing 6 distinct development utilities.

**Additional Imports**:
- `containers.arg.ModuleType` - Enumeration of module types
- `containers.arg.Arg` - Argument container with help functionality

**Conditional Execution** (lines 55-69):
```python
def run(self):
    if self.args_dict['ls'].arg_value:
        self.list_targets()
    elif self.args_dict['desc'].arg_value:
        self.desc_targets()
    elif self.args_dict['path'].arg_value:
        self.path_targets()
    elif self.args_dict['refs'].arg_value:
        self.refs_targets()
    elif self.args_dict['format'].arg_value:
        self.format_targets()
    elif self.args_dict['clean'].arg_value:
        self.clean_targets()
    else:
        Arg.get_help(ModuleType.TOOL)  # Default: show help
```

**Tool Operations**:
| Method | CLI Flag | Purpose |
|--------|----------|---------|
| `list_targets()` | `--ls` | List available build targets |
| `desc_targets()` | `--desc` | Describe target properties |
| `path_targets()` | `--path` | Show target file paths |
| `refs_targets()` | `--refs` | Show target references/dependencies |
| `format_targets()` | `--format` | Format target files |
| `clean_targets()` | `--clean` | Clean specific targets |

---

### 4.12 UpdateModuleInterface - Update Operations

**Location**: `update_module_interface.py`

**Extends**: `ModuleInterface`

**Single Operation**:
- `_update()` - Update build tools or dependencies

---

## 5. Data & Control Flow

### 5.1 Input Flow

```
CLI Arguments
     ↓
resolver/args_resolver_interface.py  (parsing)
     ↓
args_dict: dict  →  ModuleInterface.__init__()
args_resolver: ArgsResolverInterface
```

### 5.2 Execution Flow

```
ModuleInterface
     ↓ (run() called)
[Specific Interface].run()
     ↓
Template method orchestrates:
   - Abstract hook methods (implemented by concrete modules)
   - Service layer calls (BuildModuleInterface)
     ↓
Concrete Module Implementation
     ↓
services/ layer execution
```

### 5.3 Error Handling Flow

Most interfaces follow this pattern:

```python
try:
    self._operation()
except OHOSException as exception:
    raise exception  # Re-raise for upstream handling
```

**BuildModuleInterface** uses extended pattern:

```python
try:
    # Phase execution
except OHOSException as exception:
    raise exception
else:
    # Success handler (_post_target_compilation)
finally:
    # Cleanup handler (_post_build) - always executes
```

---

## 6. Integration Points

### 6.1 Upstream Integration (Callers)

| Component | Integration Type |
|-----------|-----------------|
| `modules/ohos/` | Concrete implementations of these interfaces |
| `resolver/` | Provides `ArgsResolverInterface` instances |
| `main.py` | Entry point that instantiates modules |

### 6.2 Downstream Integration (Dependencies)

| Interface | Downstream Dependencies |
|-----------|------------------------|
| `ModuleInterface` | `resolver.interface.args_resolver_interface.ArgsResolverInterface` |
| `BuildModuleInterface` | `services.interface.preload_interface.PreloadInterface`<br>`services.interface.load_interface.LoadInterface`<br>`services.interface.build_executor_interface.BuildExecutorInterface`<br>`services.interface.build_file_generator_interface.BuildFileGeneratorInterface` |
| `ToolModuleInterface` | `containers.arg.ModuleType`<br>`containers.arg.Arg` |

### 6.3 Cross-Cutting Dependencies

All interfaces depend on:
- `exceptions.ohos_exception.OHOSException` - Domain-specific exception type
- `abc.abstractmethod` - ABC decorator
- `abc.ABCMeta` - ABC metaclass (base interface only)

---

## 7. Architectural Principles

### 7.1 Inversion of Control (IoC)

The interface layer **does not instantiate** its dependencies. All dependencies are injected:

```python
# Dependencies are provided, not created
def __init__(self, args_dict: dict, args_resolver: ArgsResolverInterface):
    self._args_dict = args_dict
    self._args_resolver = args_resolver
```

### 7.2 Open/Closed Principle (OCP)

New build commands can be added by:
1. Creating a new interface extending `ModuleInterface`
2. Implementing `run()` with the new workflow
3. Adding concrete implementation in `modules/ohos/`

**No modification required** to existing interfaces.

### 7.3 Liskov Substitution Principle (LSP)

All module interfaces are substitutable for `ModuleInterface`:

```python
def execute_module(module: ModuleInterface):
    module.run()  # Works with any interface subclass

# Can pass any interface implementation
execute_module(build_module)      # BuildModuleInterface
execute_module(clean_module)      # CleanModuleInterface
execute_module(tool_module)       # ToolModuleInterface
```

---

## 8. Key Design Decisions

### 8.1 Why Abstract Methods Instead of Default Implementations?

All workflow steps use `@abstractmethod` to **force explicit implementation** by concrete classes. This prevents:
- Silent no-op behaviors
- Incomplete command implementations
- Accidental omission of required functionality

### 8.2 Why Protected Methods (Single Underscore)?

Methods use single underscore prefix (`_method_name`) to indicate:
- **Internal API** - Not intended for external callers
- **Hook Methods** - Called by the template method (`run()`)
- **Implementation Detail** - Subject to change between versions

### 8.3 Why `args_dict` Instead of Typed Arguments?

Using a dictionary for arguments provides:
- **Flexibility** - New arguments don't require interface changes
- **Extensibility** - Commands can have varying argument sets
- **Decoupling** - Interface doesn't depend on specific CLI options

### 8.4 Why `@TimerUtil.cost_time` on Build Phases?

Build operations are long-running, so timing is critical for:
- Performance monitoring
- CI/CD metrics
- User feedback on build duration
- Optimization identification

---

## 9. Usage Example

### Implementing a Custom Module

```python
from modules.interface.clean_module_interface import CleanModuleInterface
from resolver.interface.args_resolver_interface import ArgsResolverInterface

class MyCleanModule(CleanModuleInterface):
    def __init__(self, args_dict: dict, args_resolver: ArgsResolverInterface):
        super().__init__(args_dict, args_resolver)
    
    def clean_regular(self):
        # Remove build artifacts
        pass
    
    def clean_deep(self):
        # Remove all generated files
        pass
```

### Executing a Module

```python
# Instantiation (with dependency injection)
module = BuildModuleInterface(
    args_dict={'target': Arg(...)},
    args_resolver=MyArgsResolver(),
    preloader=MyPreloader(),
    loader=MyLoader(),
    target_generator=MyGenerator(),
    target_compiler=MyCompiler()
)

# Execution
module.run()  # Orchestrates all phases
```

---

## 10. Summary

The `modules/interface/` directory implements a **robust abstraction layer** that:

1. **Standardizes** command execution via the Template Method pattern
2. **Enforces** implementation contracts via Abstract Base Classes
3. **Decouples** CLI parsing from service execution via Dependency Injection
4. **Extends** functionality through inheritance without modifying existing code
5. **Measures** performance via timing decorators on expensive operations
6. **Handles** errors consistently through domain-specific exceptions

This architecture enables the HarmonyOS build system to support diverse command types (build, clean, tool, env, etc.) while maintaining a consistent execution model and clear separation of concerns.
