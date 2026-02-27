# helper/ Module Codemap

## 1. Responsibility

The `helper/` directory provides **metaclass-based utilities** for controlling class instantiation behavior and **presentation helpers** for CLI formatting. This module implements fundamental Python metaprogramming patterns that enforce architectural constraints across the HarmonyOS build system (hb).

### Core Functions:

| File | Purpose |
|------|---------|
| `singleton.py` | Enforces **single instance constraint** - ensures only one object of a class exists throughout the application lifecycle |
| `no_instance.py` | Enforces **static-only constraint** - prevents class instantiation, mandating pure static method usage |
| `separator.py` | Provides **visual formatting utilities** for CLI menu separators and section dividers |

---

## 2. Design Patterns

### 2.1 Metaclass Pattern (Python-Specific)

All three utilities leverage Python's **metaclass mechanism** - a form of class-oriented metaprogramming where the metaclass controls class creation and instance construction.

```
┌─────────────────────────────────────────────────────────────┐
│                    Metaclass Hierarchy                      │
├─────────────────────────────────────────────────────────────┤
│  type (builtin)                                             │
│    ├── Singleton (custom)  → Controls single instance       │
│    ├── NoInstance (custom) → Blocks all instantiation       │
│    └── ABCMeta (standard)  → Abstract base classes          │
└─────────────────────────────────────────────────────────────┘
```

### 2.2 Singleton Pattern

**Implementation**: `Singleton` metaclass overrides `__call__`

```python
class Singleton(type):
    _instances = {}  # Class-level registry

    def __call__(cls, *args, **kwargs):
        if cls not in cls._instances:
            cls._instances[cls] = super().__call__(*args, **kwargs)
        return cls._instances[cls]
```

**Key Characteristics**:
- **Lazy initialization**: Instance created on first access
- **Thread-safe** (in CPython due to GIL, though not explicitly synchronized)
- **Registry-based**: Uses dictionary mapping class → instance
- **Inheritance-safe**: Each class in hierarchy gets its own singleton

**Usage Pattern**:
```python
class Config(metaclass=Singleton):
    def __init__(self):
        # ... initialization

# Both references point to same object
config1 = Config()
config2 = Config()
assert config1 is config2  # True
```

### 2.3 Static Utility Class Pattern (NoInstance Metaclass)

**Implementation**: `NoInstance` metaclass overrides `__call__` to raise `TypeError`

```python
class NoInstance(type):
    def __call__(self, *args, **kwds):
        raise TypeError('This class can not be instantiation')
```

**Architectural Purpose**:
- Enforces **pure utility class** pattern
- Prevents accidental instantiation of stateless helper classes
- Compels developers to use `@staticmethod` decorators
- Runtime enforcement of static-only design

### 2.4 Factory/Builder Pattern (Separator)

**Implementation**: Simple configurable string wrapper

```python
class Separator(object):
    line = '-' * 15          # Default short separator
    long_line = '-' * 100    # Class constant for long separators

    def __init__(self, line=None):
        if line:
            self.line = f'\n{line}'  # Newline-prefixed custom label
```

**Pattern Variations**:
- **Default separator**: `Separator()` → produces `'-' * 15`
- **Labeled separator**: `Separator("CompanyName")` → produces `'\nCompanyName'`
- **Long separator**: Access via `Separator.long_line`

---

## 3. Data & Control Flow

### 3.1 Singleton Flow: Config State Management

```
┌─────────────────────────────────────────────────────────────────┐
│                     Singleton Lifecycle                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  First Call: Config()                                           │
│       │                                                         │
│       ▼                                                         │
│  Singleton.__call__(Config)                                     │
│       │                                                         │
│       ├── _instances = {} (empty)                               │
│       │                                                         │
│       ▼                                                         │
│  super().__call__() ──► Config.__init__() ──► Instance stored   │
│                                                         │       │
│  Subsequent Calls: Config()                                     │
│       │                                                         │
│       ▼                                                         │
│  Singleton.__call__(Config)                                     │
│       │                                                         │
│       └──► Return cached instance ◄─────────────────────────────┘
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 NoInstance Flow: Static Method Enforcement

```
┌─────────────────────────────────────────────────────────────────┐
│                 NoInstance Enforcement Flow                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Attempted: LogUtil()                                           │
│       │                                                         │
│       ▼                                                         │
│  NoInstance.__call__(LogUtil)                                   │
│       │                                                         │
│       └──► Raise TypeError("This class can not be instantiation")│
│                                                                 │
│  Correct Usage: LogUtil.hb_info("message")                      │
│       │                                                         │
│       ▼                                                         │
│  Direct static method invocation (bypasses __call__)           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.3 Separator Flow: CLI Menu Construction

```
┌─────────────────────────────────────────────────────────────────┐
│                   Separator Usage Flow                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Product Selection Menu (services/menu.py)                      │
│       │                                                         │
│       ├── Iterate products by company                           │
│       │       │                                                 │
│       │       └── New company detected                          │
│       │               │                                         │
│       │               ▼                                         │
│       │       Separator(company_name)  ──► "\nHuawei"          │
│       │               │                                         │
│       │               └── Inserted as visual group header      │
│       │                                                         │
│       └── List rendering checks: isinstance(choice, Separator) │
│               │                                                 │
│               ├── True: Render as non-selectable header        │
│               └── False: Render as selectable option           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.4 Control Flow Summary

| Helper | Control Point | Return Value | Consumer Pattern |
|--------|---------------|--------------|------------------|
| `Singleton` | `__call__` | Cached instance | State management |
| `NoInstance` | `__call__` | Exception raised | Static utility access |
| `Separator` | `__str__` | Formatted string | CLI rendering |

---

## 4. Integration Points

### 4.1 Dependency Graph

```
┌─────────────────────────────────────────────────────────────────┐
│                    Integration Points                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  helper/singleton.py                                            │
│       │                                                         │
│       └──► resources/config.py                                  │
│               └──► Config (global configuration singleton)     │
│                                                                 │
│  helper/no_instance.py                                          │
│       │                                                         │
│       ├──► util/log_util.py    ──► LogUtil                     │
│       ├──► util/io_util.py     ──► IoUtil                      │
│       ├──► util/system_util.py ──► SystemUtil, HandleKwargs    │
│       ├──► util/type_check_util.py ──► TypeCheckUtil           │
│       └──► util/product_util.py ──► ProductUtil                │
│                                                                 │
│  helper/separator.py                                            │
│       │                                                         │
│       ├──► main.py             ──► (imported, usage not shown) │
│       └──► services/menu.py    ──► Product selection UI        │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 Consumers Detail

#### Singleton Consumers

| Consumer | File | Purpose | Import Statement |
|----------|------|---------|------------------|
| `Config` | `resources/config.py:39` | Global build configuration registry | `from helper.singleton import Singleton` |

**Config Singleton State**:
- Build paths (`_root_path`, `_out_path`)
- Target configuration (`_board`, `_kernel`, `_product`)
- Feature flags (`_os_level`, `_target_os`, `_target_cpu`)

#### NoInstance Consumers

| Consumer | File | Purpose | Import Statement |
|----------|------|---------|------------------|
| `LogUtil` | `util/log_util.py:37` | Logging facade | `from hb.helper.no_instance import NoInstance` |
| `HandleKwargs` | `util/system_util.py:33` | Keyword argument processing | `from hb.helper.no_instance import NoInstance` |
| `SystemUtil` | `util/system_util.py:97` | System command execution | `from hb.helper.no_instance import NoInstance` |
| `IoUtil` | `util/io_util.py:29` | File I/O operations | `from hb.helper.no_instance import NoInstance` |
| `TypeCheckUtil` | `util/type_check_util.py:23` | Type validation | `from hb.helper.no_instance import NoInstance` |
| `ProductUtil` | `util/product_util.py:30` | Product metadata queries | `from hb.helper.no_instance import NoInstance` |

**Common Pattern**:
```python
class XxxUtil(metaclass=NoInstance):
    @staticmethod
    def operation():
        # Pure function implementation
```

#### Separator Consumers

| Consumer | File | Purpose | Import Statement |
|----------|------|---------|------------------|
| `Menu` | `services/menu.py:38` | Interactive product selection UI | `from helper.separator import Separator` |
| `Main` | `main.py:89` | CLI initialization | `from helper.separator import Separator` |

**Menu Integration**:
```python
# services/menu.py:100
product_key = Separator(company_separator)
# Creates visual separator with company name as header
```

### 4.3 Import Path Variations

Two import conventions are used across the codebase:

| Convention | Example | Context |
|------------|---------|---------|
| Relative from hb | `from hb.helper.no_instance import NoInstance` | Used in `util/` subdirectory |
| Absolute from helper | `from helper.singleton import Singleton` | Used in `resources/` subdirectory |
| Absolute from helper | `from helper.separator import Separator` | Used in `main.py`, `services/` |

### 4.4 Architectural Constraints Enforced

| Metaclass | Constraint | Violation Consequence |
|-----------|------------|----------------------|
| `Singleton` | One instance per class | Multiple calls return same object (no violation possible) |
| `NoInstance` | No instantiation allowed | `TypeError` at runtime if instantiation attempted |

---

## Summary

The `helper/` module is a **foundational infrastructure layer** providing Python metaprogramming utilities:

1. **`Singleton`**: Enables global state management for configuration
2. **`NoInstance`**: Enforces static utility class pattern across utility modules
3. **`Separator`**: Facilitates CLI UX with visual grouping elements

These helpers demonstrate sophisticated use of Python's metaclass system to enforce architectural constraints at runtime, ensuring consistent design patterns across the HarmonyOS build system.
