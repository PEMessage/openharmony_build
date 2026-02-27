# hb/__main__.py - Codemap

## 1. Responsibility

**`hb/__main__.py`** serves as the **entry point bootstrapper** for the HarmonyOS build system CLI (`hb`). Its primary responsibilities include:

- **Workspace Validation**: Detects whether the current working directory is within a valid HarmonyOS source tree by traversing upward to locate `build/hb/resources/global_var.py`.
- **Dynamic Module Loading**: Uses Python's `importlib` to dynamically load and execute the actual `main.py` module located at `<ohos_root>/build/hb/main.py`.
- **Context Switching**: Bridges the gap between the package-level entry point (`python -m hb`) and the project-specific build logic by:
  - Computing the relative path from current directory to OHOS root
  - Injecting the correct `main.py` location into the Python module namespace
- **Error Handling**: Raises descriptive exceptions when invoked outside the OHOS source directory.

> **Note**: This file is the **CLI facade** that delegates all actual build logic to `main.py`. It enables `hb` to be both a standalone Python package and context-aware within the OHOS source tree.

---

## 2. Design Patterns

### 2.1 Facade Pattern
The `__main__.py` acts as a simplified interface to the complex subsystem in `main.py`. It abstracts away:
- Module path resolution
- Dynamic loading mechanics  
- Workspace context detection

### 2.2 Dynamic Module Loading (Importlib Pattern)
```python
spec = importlib.util.spec_from_file_location('main', entry_path)
api = importlib.util.module_from_spec(spec)
spec.loader.exec_module(api)
```
Uses `importlib.util` for programmatic module loading, avoiding hardcoded imports that would fail outside the OHOS tree context.

### 2.3 Directory Traversal (Path Resolution Pattern)
```python
cur_dir = os.getcwd()
while cur_dir != "/":
    global_var = os.path.join(cur_dir, 'build', 'hb', 'resources', 'global_var.py')
    if os.path.exists(global_var):
        return True, cur_dir
    cur_dir = os.path.dirname(cur_dir)
```
Implements **upward directory traversal** to discover the OHOS root, enabling `hb` to work from any subdirectory within the project.

### 2.4 Singleton-Style Entry Point
The `main()` function instantiates `Main` class (from `main.py`) and invokes its `main()` method, effectively delegating to a singleton coordinator.

---

## 3. Data & Control Flow

### 3.1 Execution Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│  User executes: python -m hb <command> [options]                    │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  hb/__main__.py executes                                          │
│  1. is_in_ohos_dir() - traverses upward searching for:            │
│     build/hb/resources/global_var.py                               │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
        ┌─────────────────────┐    ┌─────────────────────┐
        │  NOT FOUND          │    │  FOUND              │
        │  Raise Exception    │    │  ohos_root_path set │
        │  [OHOS_ERROR]       │    │                     │
        └─────────────────────┘    └──────────┬──────────┘
                                              │
                                              ▼
┌─────────────────────────────────────────────────────────────────────┐
│  2. Compute entry_path to:                                          │
│     <ohos_root>/build/hb/main.py                                    │
│                                                                     │
│  3. Dynamic Loading via importlib:                                  │
│     - spec_from_file_location()                                     │
│     - module_from_spec()                                            │
│     - exec_module()                                                 │
└────────────────────────────────┬────────────────────────────────────┘
                                 │
                                 ▼
┌─────────────────────────────────────────────────────────────────────┐
│  4. Instantiate and delegate:                                       │
│     main_api = api.Main()                                           │
│     main_api.main()  <-- Control transfers to main.py               │
└─────────────────────────────────────────────────────────────────────┘
```

### 3.2 Data Flow

| Step | Function | Input | Output | Description |
|------|----------|-------|--------|-------------|
| 1 | `is_in_ohos_dir()` | `os.getcwd()` | `(bool, str)` | Validates workspace context |
| 2 | Path resolution | `ohos_root_path` | `entry_path` | Constructs absolute path to main.py |
| 3 | Module loading | `entry_path` | `api` module | Dynamically loads main.py as module |
| 4 | Delegation | N/A | Exit code | Invokes `Main.main()` from loaded module |

### 3.3 Error Handling Flow

```
┌────────────────────────────────────────────────────────────────────┐
│  is_in_ohos_dir() returns (False, '')                              │
└──────────────────────────────┬─────────────────────────────────────┘
                               │
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│  raise Exception(                                                  │
│    "[OHOS_ERROR]: Please call hb utilities inside ohos source      │
│     directory"                                                      │
│  )                                                                  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 4. Integration Points

### 4.1 Upstream Dependencies (Imports)

| Import | Purpose |
|--------|---------|
| `os` | Directory traversal, path manipulation |
| `sys` | Exit handling (`sys.exit()`) |
| `importlib` | Dynamic module loading |
| `importlib.util` | Programmatic module spec creation |

### 4.2 Downstream Dependencies (Invokes)

| Component | Path | Role |
|-----------|------|------|
| `main.py` | `<ohos_root>/build/hb/main.py` | Core build orchestrator |
| `Main` class | Defined in `main.py` | Build system coordinator |
| `Main.main()` | Entry method | CLI command router |

### 4.3 Project Structure Context

```
<ohos_root>/
├── build/
│   └── hb/                          # hb package root
│       ├── __main__.py              # ← ENTRY POINT (this file)
│       ├── main.py                  # ← Core logic (delegates here)
│       ├── hb/                      # Package namespace
│       │   └── __main__.py          # Nested entry (unused)
│       ├── modules/                 # Build module implementations
│       ├── services/                # Build services (GN, Ninja, etc.)
│       ├── resolver/                # Argument resolvers
│       ├── containers/              # Data containers (Arg, Status)
│       └── resources/               # Configs, args JSON files
│           └── global_var.py        # Workspace validation marker
```

### 4.4 External Integration Points

| External Component | Integration Type | Description |
|-------------------|------------------|-------------|
| HarmonyOS Source Tree | File System | Requires specific directory structure |
| `global_var.py` | Marker File | Validates OHOS workspace |
| `main.py` | Module Import | Runtime-loaded build orchestrator |

---

## 5. Key Functions Reference

### `is_in_ohos_dir() -> Tuple[bool, str]`
- **Line**: 25-33
- **Returns**: `(in_ohos: bool, ohos_root_path: str)`
- **Algorithm**: Upward directory traversal from `os.getcwd()` to root (`/`)
- **Marker**: Checks existence of `build/hb/resources/global_var.py`

### `main() -> None`
- **Line**: 36-48
- **Decorator**: None (top-level entry)
- **Flow**:
  1. Validate workspace context
  2. Compute relative path to main.py
  3. Dynamic import via `importlib`
  4. Instantiate `Main` and delegate

---

## 6. Usage Patterns

### 6.1 Valid Usage (Inside OHOS Tree)
```bash
cd /path/to/ohos/
python -m hb build --product <product_name>
# or
hb build --product <product_name>
```

### 6.2 Invalid Usage (Outside OHOS Tree)
```bash
cd /tmp
python -m hb build
# Raises: [OHOS_ERROR]: Please call hb utilities inside ohos source directory
```

---

## 7. Architecture Notes

- **Purposeful Decoupling**: This file intentionally contains minimal logic to avoid import errors when `hb` is installed as a standalone package but executed outside the OHOS context.
- **Context Injection**: By dynamically loading `main.py` with the correct OHOS root context, the build system can resolve relative paths correctly from any subdirectory.
- **Entry Point Convention**: The `if __name__ == "__main__": sys.exit(main())` pattern ensures proper exit code propagation to the shell.
