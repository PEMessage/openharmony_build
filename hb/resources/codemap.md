# resources/ - Configuration and Global State

Centralized configuration management, argument definitions, and global state for the HarmonyOS build system. Implements the Registry and Singleton patterns for build-wide configuration persistence.

## Responsibility

**Primary Purpose:** Provide configuration infrastructure and shared state for the hb build system.

**Key Responsibilities:**
- **Global Configuration (`config.py`)**: Singleton pattern for build configuration persistence
- **Global Variables (`global_var.py`)**: Module-level constants and paths (status files, log paths, build directories)
- **Argument Definitions (`args/`)**: JSON-driven CLI argument specifications for all 11 commands
- **Status Definitions (`status/`)**: Error code taxonomy and error message registry
- **Build Tool Configurations (`build_tools/`)**: External tool paths and templates

## Design Patterns

### 1. **Singleton Pattern** (`config.py`)
The `Config` class uses `Singleton` metaclass from `helper/singleton.py` to ensure single global instance:
```python
class Config(metaclass=Singleton):
    """Global configuration registry."""
```
Used by 40+ files to share build state (product, variant, build directory, etc.).

### 2. **Registry Pattern** (`global_var.py`)
Module-level constants act as a global registry:
- `STATUS_FILE`: Error taxonomy location
- `DEFAULT_BUILD_LOG`: Build log path
- `DEFAULT_BUILD_DIR`: Default output directory
- `PREBUILTS_DOWNLOAD_SCRIPT`: SDK download script path

### 3. **Configuration-as-Data** (`args/*.json`)
Arguments defined declaratively in JSON, loaded at runtime:
```json
{
  "args": [
    {
      "arg_name": "product",
      "arg_type": "str",
      "arg_value": "",
      "resolve_function": "resolve_product",
      "help_info": "Build a named product"
    }
  ]
}
```

### 4. **Externalized Metadata** (`status/status.json`)
Error codes, messages, and solutions stored externally:
```json
{
  "error_code": "1000",
  "error_type": "Preloader Error",
  "description": "Failed to preload configuration",
  "solution": "Check product configuration"
}
```

## Data & Control Flow

### Configuration Flow

```
1. Application Startup
   └─> Config() instantiated (singleton)

2. Argument Parsing (resolver/)
   ├─> Load args/*.json definitions
   ├─> Build Arg objects via Arg.create_instance_by_dict()
   └─> Store in resolver._args_to_function registry

3. Build Execution
   ├─> Config.set_product(), Config.set_variant()
   ├─> Services query Config.get_*() for current state
   └─> Cross-phase state persistence via JSON files

4. Error Handling
   └─> Lookup error metadata from status.json by error_code
```

### Configuration Persistence Mechanism

Arguments and state can be persisted as JSON files in the build output directory:
```
out/{product}/build_configs/
├── args/           # Serialized Arg objects
├── status/         # Error state
└── tools/          # Tool configurations
```

## Integration Points

### Core Python Files

| File | Pattern | Consumers |
|------|---------|-----------|
| `config.py` | Singleton | 40+ files (modules/, services/, util/, resolver/) |
| `global_var.py` | Module constants | Used for paths across all layers |

### JSON Configuration Directories

| Directory | Contents | Purpose |
|-----------|----------|---------|
| `args/default/` | Argument definitions for 11 commands | CLI argument specification |
| `build_tools/` | External tool configs | HPM, Node.js, prebuilt paths |
| `status/` | Error taxonomy | Error code → message/solution mapping |

### Dependency Injection Consumers

- **resolvers/**: Load argument definitions from JSON
- **services/**: Query Config singleton for current product/variant
- **modules/**: Store persistent state via Config
- **exceptions/**: Lookup error metadata from status.json

## Key Files

### Python Modules
- **`config.py`**: Global configuration singleton with get/set methods for product, variant, build directory, and custom arguments
- **`global_var.py`**: Module-level constants defining paths to status files, log files, prebuilt download scripts, and default directories

### JSON Configurations
- **`args/default/buildargs.json`**: Build command argument definitions (product, variant, jobs, etc.)
- **`args/default/cleanargs.json`**: Clean command arguments
- **`args/default/setargs.json`**: Set command arguments
- **`args/default/toolargs.json`**: Tool execution arguments
- **`status/status.json`**: 4-digit error code taxonomy (0000-4000) organized by build phase
- **`build_tools/ohos_build_tools.json`**: HPM and Node.js tool paths

## Error Code Taxonomy

| Code Range | Category | Description |
|------------|----------|-------------|
| 0000-0999 | General | System-level errors |
| 1000-1999 | Preloader | Product configuration errors |
| 2000-2999 | Loader | Subsystem loading errors |
| 3000-3999 | GN | Build file generation errors |
| 4000-4999 | Ninja | Compilation errors |

See also:
- [containers/codemap.md](../containers/codemap.md) - Arg value objects
- [exceptions/codemap.md](../exceptions/codemap.md) - OHOSException usage
- [resolver/codemap.md](../resolver/codemap.md) - Argument resolution flow
