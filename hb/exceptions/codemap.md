# CodeMap: exceptions/ Module

## Overview

| Attribute | Value |
|-----------|-------|
| **Directory** | `/home/zhuojw/a_git/ohos-mani-v2/build/hb/exceptions/` |
| **Primary Module** | `ohos_exception.py` |
| **Purpose** | Centralized exception handling framework for the HarmonyOS build system (hb) |
| **Architecture Pattern** | Hierarchical Exception Class + Externalized Error Metadata |

---

## 1. Responsibility

The `exceptions/` module defines the **foundational error handling infrastructure** for the hb build system:

1. **Base Exception Definition**: Provides `OHOSException`, the root exception class for all hb-specific errors
2. **Error Code System**: Implements a numeric error code taxonomy (4-digit codes) organized by compilation phase
3. **Externalized Error Metadata**: Decouples error messages, types, descriptions, and solutions from code via JSON configuration
4. **Integration with Logging**: Works with `containers/status.py` to provide formatted error reporting and diagnostic output

### Error Code Taxonomy

| Code Range | Compilation Stage | Description |
|------------|-------------------|-------------|
| `0000-0999` | General/Pre-build | Configuration, argument validation, I/O errors |
| `1000-1999` | Preloader | Preloader phase errors |
| `2000-2999` | Loader | Loader phase errors (subsystem/component loading) |
| `3000-3999` | GN | GN meta-build generation errors |
| `4000-4999` | Ninja | Ninja build execution errors |

---

## 2. Design Patterns

### 2.1 Hierarchical Exception Architecture

```
BaseException (Python built-in)
    └── Exception (Python built-in)
            └── OHOSException (hb root exception)
                    └── [Implicit: All hb-specific errors]
```

**Pattern**: The module uses a **flat exception hierarchy** with a single base class. Rather than creating many subclasses, it uses **error code discrimination** to identify error types.

### 2.2 Externalized Error Metadata (Separation of Concerns)

The `OHOSException` class implements a **Data-Driven Error Resolution** pattern:

- Error metadata (type, description, solution) is stored externally in `resources/status/status.json`
- Error instances are identified by numeric `code` parameter
- Runtime lookup provides human-readable error context

**Benefits**:
- Error messages can be updated without code changes
- Supports localization
- Centralized error documentation

### 2.3 Decorator-Based Exception Translation

The companion module `containers/status.py` provides `@throw_exception` decorator implementing **Exception Translation**:

- Catches both `OHOSException` and generic `Exception`
- Normalizes all exceptions to a consistent error format
- Handles exit code propagation (`exit(-1)`)
- Provides formatted stack trace logging

### 2.4 Singleton Error Configuration

`STATUS_FILE` path is defined in `resources/global_var.py`:
```python
STATUS_FILE = os.path.join(CURRENT_HB_DIR, 'resources/status/status.json')
```

---

## 3. Data & Control Flow

### 3.1 Exception Propagation Flow

```
┌─────────────────────────────────────────────────────────────────────┐
│                         Exception Propagation                        │
└─────────────────────────────────────────────────────────────────────┘

  [Error Detected]
         │
         ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  raise           │────▶│  @throw_exception │────▶│  status.json     │
│  OHOSException(  │     │  decorator        │     │  lookup          │
│    message,      │     │  (containers/     │     │  (get_solution/  │
│    code)         │     │   status.py)      │     │   get_type/      │
└──────────────────┘     └──────────────────┘     │   get_desc)      │
                                                  └──────────────────┘
                                                           │
         ┌──────────────────────────────────────────────────┘
         ▼
┌──────────────────┐     ┌──────────────────┐     ┌──────────────────┐
│  LogUtil.write_  │────▶│  build.log       │     │  Formatted       │
│  log()           │     │  (out/build.log) │────▶│  stderr output   │
└──────────────────┘     └──────────────────┘     └──────────────────┘
                                                           │
                                                           ▼
                                                  ┌──────────────────┐
                                                  │  exit(-1)        │
                                                  └──────────────────┘
```

### 3.2 Error Code Resolution Flow

```python
# Error metadata lookup sequence:
raise OHOSException("message", "2001")
         │
         ▼
OHOSException.__init__(message, code=2001)
         │
         ▼
get_solution() ──▶ open(STATUS_FILE) ──▶ json.load()
get_type()     ──▶ status_file[str(code)]['type']
get_desc()     ──▶ status_file[str(code)]['description']
```

### 3.3 Status.json Schema

```json
{
  "CODE": {
    "code": "CODE",
    "type": "Error category/type",
    "pattern": "Regex pattern for auto-detection",
    "description": "Human-readable description",
    "solution": "Remediation steps (string or array)"
  }
}
```

---

## 4. Integration Points

### 4.1 Module Dependencies (Inbound)

| Consumer Module | Import Location | Usage Pattern |
|-----------------|-----------------|---------------|
| `main.py` | Line 39 | Module validation errors |
| `containers/arg.py` | Line 48 | Argument type validation |
| `containers/status.py` | Line 22 | Exception handling decorator |
| `modules/ohos_*_module.py` (8 files) | Various | Module-specific errors |
| `modules/interface/*_interface.py` (6 files) | Various | Interface error handling |
| `resolver/*_args_resolver.py` (8 files) | Various | Argument resolution errors |
| `resolver/args_factory.py` | Line 21 | Argument factory errors |
| `services/gn.py` | Line 28 | GN service errors |
| `services/ninja.py` | Line 23 | Ninja execution errors |
| `services/loader.py` | Line 24 | Loader service errors |
| `services/hpm.py` | Line 28 | HPM (HarmonyOS Package Manager) errors |
| `services/menu.py` | Line 36 | Interactive menu errors |
| `services/hdc.py` | Line 28 | HDC (HarmonyOS Device Connector) errors |
| `util/io_util.py` | Line 26 | I/O utility errors |
| `util/log_util.py` | Line 27 | Logging utility errors |
| `util/device_util.py` | Line 21 | Device utility errors |
| `util/component_util.py` | Line 26 | Component utility errors |
| `util/product_util.py` | Line 23 | Product utility errors |
| `util/loader/*.py` (4 files) | Various | Loader utility errors |
| `util/prebuild/patch_process.py` | Line 21 | Patch processing errors |
| `util/type_check_util.py` | Line 19 | Type checking errors |

**Total Consumers**: ~52 import sites across 35+ files

### 4.2 Dependencies (Outbound)

| Dependency | Purpose |
|------------|---------|
| `json` | Parsing status.json error metadata |
| `resources.global_var.STATUS_FILE` | Path to error configuration file |

### 4.3 Key Integration Patterns

#### Module Pattern (Try-Except-Reraise)
```python
# Common pattern across modules
from exceptions.ohos_exception import OHOSException

def some_operation():
    try:
        # ... operation ...
    except SomeException as e:
        raise OHOSException(f"Context: {e}", "XXXX")
```

#### Resolver Pattern (Validation)
```python
# Common pattern in args resolvers
if invalid_condition:
    raise OHOSException('ERROR argument "--param": Invalid value', "CODE")
```

#### Decorator Pattern (Global Handler)
```python
from containers.status import throw_exception

@throw_exception
def entry_point():
    # All exceptions caught, formatted, logged
    raise OHOSException("...", "...")
```

---

## 5. Class Reference

### 5.1 OHOSException

**Location**: `exceptions/ohos_exception.py:24`

```python
class OHOSException(Exception):
    """
    Root exception class for HarmonyOS build system errors.
    
    Attributes:
        _code (int): Numeric error code for external metadata lookup
        _message (str): Human-readable error message
    """
    
    def __init__(self, message: str, code: int = 0):
        """
        Initialize exception with message and optional error code.
        
        Args:
            message: Human-readable error description
            code: Numeric error code (default: 0 = unknown)
        """
    
    def get_solution(self) -> str:
        """
        Retrieve solution text from status.json.
        Returns 'UNKNOWN REASON' if code not found.
        """
    
    def get_type(self) -> str:
        """
        Retrieve error type/category from status.json.
        Returns 'UNKNOWN ERROR TYPE' if not found.
        """
    
    def get_desc(self) -> str:
        """
        Retrieve detailed description from status.json.
        Returns 'NO DESCRIPTION' if not found.
        """
```

---

## 6. Error Code Registry (Selected)

| Code | Type | Description |
|------|------|-------------|
| `0000` | Unknown | Generic unknown error |
| `0001` | Prebuilts | Missing prebuilt dependencies |
| `0008` | I/O | File not found (io_util) |
| `0019` | Config | Development environment not initialized |
| `1001` | Preloader | Preloader result incorrect |
| `2001-2014` | Loader | Various loader phase errors |
| `3000-3014` | GN | GN meta-build errors (syntax, missing files, deps) |
| `4000-4016` | Ninja | Build execution errors (syntax, linking, undefined refs) |

---

## 7. Design Characteristics

### Strengths
1. **Centralized Error Management**: Single source of truth for error handling
2. **Extensible Code System**: Numeric ranges allow new error categories
3. **Decoupled Metadata**: Error solutions updated without code changes
4. **Consistent UX**: Uniform error formatting across all modules

### Considerations
1. **File I/O on Exception**: Each metadata lookup opens/reads/parses JSON
2. **String-based Code**: Error codes passed as strings ("2001" not 2001)
3. **Flat Hierarchy**: No semantic exception subclasses for catch-specific handling

---

## 8. Related Files

| File | Relationship |
|------|--------------|
| `containers/status.py` | Exception handling decorator and error output formatting |
| `resources/global_var.py` | STATUS_FILE path constant |
| `resources/status/status.json` | Error metadata database |
| `util/log_util.py` | Log writing for error traces |
