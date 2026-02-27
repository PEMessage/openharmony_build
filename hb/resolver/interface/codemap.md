# Codemap: resolver/interface/

## 1. Responsibility

The `resolver/interface/` directory defines the **abstract contract** for argument resolution within the HarmonyOS build system (hb). It establishes a standardized interface that enables polymorphic argument processing across different build commands (build, set, clean, env, tool, package, publish, install, update, push, indep_build).

**Core Purpose:**
- Define the `ArgsResolverInterface` abstract base class (ABC) that all concrete resolvers must implement
- Establish a **dynamic dispatch mechanism** mapping argument names to their corresponding resolution functions
- Provide a **template method** for argument resolution with consistent error handling
- Decouple argument parsing from argument resolution logic

---

## 2. Design Patterns

### 2.1 Abstract Base Class (ABC) Pattern

```python
class ArgsResolverInterface(metaclass=ABCMeta):
```

The interface uses Python's `abc.ABCMeta` metaclass to enforce:
- **Inheritance Contract**: All concrete resolver implementations must inherit from `ArgsResolverInterface`
- **Method Signature Enforcement**: Subclasses must implement resolution methods referenced in arg configurations

### 2.2 Template Method Pattern

The interface implements a template method structure with:

| Method | Role | Description |
|--------|------|-------------|
| `__init__(args_dict)` | Template | Initializes the resolver by mapping args to functions |
| `_map_args_to_function()` | Primitive Operation | Validates and binds argument names to callable resolution methods |
| `resolve_arg()` | Template Entry | Public API for resolving a specific argument |

**Execution Flow:**
```
ConcreteResolver.__init__(args_dict)
  └── super().__init__(args_dict) [ArgsResolverInterface.__init__]
        └── _map_args_to_function(args_dict)
              ├── Validates each Arg has a corresponding method
              ├── Binds Arg.resolve_function → instance method
              └── Stores in self._args_to_function[arg_name] = method
```

### 2.3 Registry Pattern

The `_args_to_function` dictionary acts as a **method registry**:
```python
self._args_to_function = dict()  # Key: arg_name, Value: bound method
```

This enables O(1) lookup of resolution functions at runtime.

### 2.4 Strategy Pattern (via Function Binding)

Each `Arg` specifies its resolution strategy via `resolve_function` string:
```python
# Arg configuration in JSON
{
    "arg_name": "--product-name",
    "resolve_function": "resolve_product_name"
}
```

The interface binds these strings to actual methods, allowing **declarative strategy selection**.

---

## 3. Data & Control Flow

### 3.1 Data Structures

#### Arg Container (`containers.arg.Arg`)
| Attribute | Type | Purpose |
|-----------|------|---------|
| `arg_name` | str | Normalized argument name (e.g., `product_name`) |
| `arg_value` | Any | Parsed CLI value |
| `arg_phase` | BuildPhase/[] | Build phase(s) when arg is resolved |
| `arg_type` | ArgType | BOOL, INT, STR, LIST, DICT, SUBPARSERS |
| `resolve_function` | str | Name of resolution method to invoke |
| `arg_attribute` | dict | Additional metadata (deprecated, optional, abbreviation) |

#### Internal State
```python
self._args_to_function: dict  # {arg_name: bound_method}
```

### 3.2 Resolution Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 1: Initialization (Constructor)                              │
├─────────────────────────────────────────────────────────────────────┤
│  ArgsResolverInterface.__init__(args_dict: dict)                    │
│    ├── Create empty _args_to_function registry                      │
│    └── Call _map_args_to_function(args_dict)                        │
│         ├── Iterate args_dict.values()                              │
│         ├── Skip 'sshkey' arg (security exception)                  │
│         ├── Verify hasattr(self, function_name)                     │
│         ├── Verify callable(getattr(self, function_name))           │
│         ├── Bind: entity.resolve_function = bound_method            │
│         └── Register: _args_to_function[args_name] = bound_method   │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Phase 2: Runtime Resolution                                        │
├─────────────────────────────────────────────────────────────────────┤
│  resolve_arg(target_arg: Arg, module)                               │
│    ├── Check: target_arg.arg_name in _args_to_function              │
│    │   └── Raise OHOSException(0000) if not found                   │
│    ├── Check: callable(_args_to_function[arg_name])                 │
│    │   └── Raise OHOSException if not callable                      │
│    ├── Retrieve: resolve_function = _args_to_function[arg_name]     │
│    └── Invoke: return resolve_function(target_arg, module)          │
│         └── Concrete resolver's static method executes              │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.3 Error Handling Flow

The `@throw_exception` decorator (from `containers.status`) wraps both public and private methods:

```python
@throw_exception
def resolve_arg(self, target_arg: Arg, module): ...

@throw_exception
def _map_args_to_function(self, args_dict: dict): ...
```

**Error Codes:**
| Code | Context | Description |
|------|---------|-------------|
| 0000 | `resolve_arg` | Resolution function not defined for argument |
| 0004 | `_map_args_to_function` | No resolution function exists for arg |

---

## 4. Integration Points

### 4.1 Upstream Dependencies

| Component | Relationship | Purpose |
|-----------|--------------|---------|
| `containers.arg.Arg` | Data container | Argument definition and value storage |
| `containers.status.throw_exception` | Decorator | Exception handling and logging |
| `exceptions.ohos_exception.OHOSException` | Error type | Structured error reporting |

### 4.2 Downstream Implementations

**12 Concrete Resolver Classes** inherit from `ArgsResolverInterface`:

| Implementation | Module Type | Key Resolution Methods |
|----------------|-------------|------------------------|
| `BuildArgsResolver` | BUILD | `resolve_product`, `resolve_target`, `clean_output_if_needed` |
| `SetArgsResolver` | SET | `resolve_product_name`, `resolve_set_parameter` |
| `CleanArgsResolver` | CLEAN | `resolve_clean`, `clean_output` |
| `EnvArgsResolver` | ENV | `resolve_env`, `resolve_log_level` |
| `ToolArgsResolver` | TOOL | `resolve_tool` |
| `IndepBuildArgsResolver` | INDEP_BUILD | `resolve_target`, `resolve_product` |
| `PackageArgsResolver` | PACKAGE | `resolve_package` |
| `PublishArgsResolver` | PUBLISH | `resolve_publish` |
| `InstallArgsResolver` | INSTALL | `resolve_install` |
| `UpdateArgsResolver` | UPDATE | `resolve_update` |
| `PushArgsResolver` | PUSH | `resolve_push` |
| `ArgsResolver` (judge) | BUILD | `resolve_target` |

### 4.3 Integration Architecture

```
                    ┌─────────────────────────────────────┐
                    │    resolver/interface/              │
                    │  ┌─────────────────────────────┐    │
                    │  │ ArgsResolverInterface (ABC) │    │
                    │  │  - _args_to_function: dict  │    │
                    │  │  - resolve_arg()            │    │
                    │  │  - _map_args_to_function()  │    │
                    │  └─────────────────────────────┘    │
                    └─────────────────┬───────────────────┘
                                      │ inherits
        ┌─────────────────────────────┼─────────────────────────────┐
        │                             │                             │
        ▼                             ▼                             ▼
┌───────────────────┐    ┌───────────────────┐    ┌───────────────────┐
│ BuildArgsResolver │    │ SetArgsResolver   │    │ CleanArgsResolver │
│ ───────────────── │    │ ───────────────── │    │ ───────────────── │
│ resolve_product() │    │ resolve_product_  │    │ resolve_clean()   │
│ resolve_target()  │    │   name()          │    │ ...               │
│ ...               │    │ resolve_set_      │    │                   │
│                   │    │   parameter()     │    │                   │
└───────────────────┘    └───────────────────┘    └───────────────────┘
        │                             │                             │
        └─────────────────────────────┼─────────────────────────────┘
                                      │ instantiates
                                      ▼
                    ┌─────────────────────────────────────┐
                    │       modules/ (Build System)       │
                    │  - BuildModuleInterface             │
                    │  - SetModuleInterface               │
                    │  - CleanModuleInterface             │
                    │  - ...                              │
                    └─────────────────────────────────────┘
```

### 4.4 Resolution Method Contract

All resolution methods must follow this signature:
```python
@staticmethod
def resolve_<arg_name>(target_arg: Arg, module: ModuleInterface) -> Any:
    """
    Resolve the argument value and apply side effects to the module.
    
    Args:
        target_arg: The Arg instance containing name, value, and metadata
        module: The build module interface for accessing build services
    
    Returns:
        Resolution result (typically None; side effects on module)
    """
```

### 4.5 Configuration-Driven Binding

Argument-to-method binding is **declarative** via JSON configuration files:

```json
{
    "product_name": {
        "arg_name": "--product-name",
        "resolve_function": "resolve_product_name",
        "arg_phase": "prebuild",
        "arg_type": "str"
    }
}
```

The `Arg.create_instance_by_dict()` factory method parses these configurations and the interface validates that referenced `resolve_function` methods exist on the concrete resolver class.

---

## 5. Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| **Static methods for resolvers** | Resolution logic is stateless; avoids instance coupling |
| **SSH key exclusion** | Security: `sshkey` arg is explicitly excluded from resolution mapping |
| **Runtime method validation** | Fail-fast: Missing resolution functions detected at init, not runtime |
| **ModuleInterface parameter** | Dependency injection: Allows resolvers to interact with build system services |
| **String-to-method binding** | Declarative configuration enables adding args without code changes |

---

## 6. File Reference

| File | Lines | Purpose |
|------|-------|---------|
| `args_resolver_interface.py` | 56 | Abstract base class defining resolver contract |

---

*Generated for HarmonyOS hb build system - Interface layer for argument resolution*
