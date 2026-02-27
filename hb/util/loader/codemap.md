# util/loader/ - Subsystem and Target Loading

Build configuration loading utilities that parse product definitions, subsystem configurations, and generate GN build files. Implements the Loader stage of the build pipeline.

## Responsibility

**Primary Purpose:** Parse OHOS build configuration files and generate GN (Generate Ninja) build files.

**Key Responsibilities:**
- **Target Generation (`generate_targets_gn.py`)**: Generate .gni files defining parts, inner_kits, system_kits
- **Bundle Loading (`load_bundle_file.py`)**: Parse bundle.json files for component metadata
- **Build Configuration (`load_ohos_build.py`)**: Parse ohos.build files defining subsystems and parts
- **Platform Loading (`platforms_loader.py`)**: Load platform-specific build configurations
- **Subsystem Scanning (`subsystem_scan.py`)**: Scan source tree for subsystem definitions
- **Subsystem Info (`subsystem_info.py`)**: Query and manage subsystem metadata
- **Platform Merge (`merge_platform_build.py`)**: Merge platform-specific build configurations

## Design Patterns

### 1. **Template Method Pattern**
`generate_targets_gn.py` uses Jinja2 templates for code generation:
```python
PARTS_LIST_GNI_TEMPLATE = """
parts_list = [
  {}
]
"""
```

### 2. **Parser Pattern**
Multiple parsers for different config formats:
- `load_ohos_build.py`: Parses BUILD.gn and ohos.build files
- `load_bundle_file.py`: Parses bundle.json (HPM package metadata)
- `platforms_loader.py`: Parses platform configuration JSONs

### 3. **Registry Pattern**
`subsystem_info.py` maintains registry of loaded subsystems and their metadata.

## Data & Control Flow

### Loader Pipeline Flow

```
1. Service Layer invokes Loader
   └─> services/loader.py calls loader utilities

2. Configuration Parsing
   ├─> subsystem_scan.py: Scan for subsystem directories
   ├─> load_ohos_build.py: Parse ohos.build files
   ├─> load_bundle_file.py: Parse bundle.json for components
   └─> platforms_loader.py: Load platform configs

3. Data Processing
   ├─> subsystem_info.py: Build subsystem metadata registry
   ├─> merge_platform_build.py: Merge platform-specific configs
   └─> generate_targets_gn.py: Generate .gni output files

4. Output Generation
   └─> Write parts_list.gni, inner_kits.gni, system_kits.gni
```

## Integration Points

### Key Files

| File | Purpose | Output |
|------|---------|--------|
| `generate_targets_gn.py` | GN file generation | .gni files for build system |
| `load_ohos_build.py` | Parse ohos.build | Subsystem/part definitions |
| `load_bundle_file.py` | Parse bundle.json | HPM component metadata |
| `platforms_loader.py` | Load platform configs | Platform-specific settings |
| `subsystem_scan.py` | Scan source tree | Subsystem discovery |
| `subsystem_info.py` | Subsystem registry | Metadata queries |
| `merge_platform_build.py` | Config merging | Unified build config |

### Upstream Dependencies
- **resources/config.py**: Global configuration singleton
- **resources/global_var.py**: Path constants
- **util/log_util.py**: Logging

### Downstream Consumers
- **services/loader.py**: Primary consumer
- **GN build system**: Consumes generated .gni files

## Key Technical Details

### Jinja2 Templates
Uses Jinja2 from `third_party/jinja2` for code generation:
```python
from jinja2 import Template
```

### Output Files
Generated files written to `out/{product}/build_configs/`:
- `parts_list.gni`: List of parts to build
- `inner_kits.gni`: Internal API dependencies
- `system_kits.gni`: System API dependencies

### Configuration Sources
- `build/subsystem_config.json`: Subsystem definitions
- `vendor/{vendor}/{product}/config.json`: Product configuration
- `{component}/bundle.json`: Component metadata
- `{component}/ohos.build`: Build rules

See also:
- [../services/loader.py](../services/loader.py) - Service wrapper
- [../preloader/](../preloader/) - Pre-build configuration stage