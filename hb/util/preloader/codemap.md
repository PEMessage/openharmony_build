# util/preloader/ - Pre-Build Configuration

Preloader utilities for parsing product configurations, generating build_config.json, and preparing the build environment. This is the first stage of the build pipeline.

## Responsibility

**Primary Purpose:** Parse product and subsystem configurations to generate the build configuration file.

**Key Responsibilities:**
- **Vendor/Product Config Parsing (`parse_vendor_product_config.py`)**: Parse vendor and product JSON configs
- **Lite Subsystems Parsing (`parse_lite_subsystems_config.py`)**: Parse subsystem configs for lite devices
- **Data Processing (`preloader_process_data.py`)**: Process and merge all configuration data
- **Build Config Generation**: Generate build_config.json for downstream stages

## Design Patterns

### 1. **Parser Chain Pattern**
Multiple specialized parsers for different config types:
```
parse_vendor_product_config.py → Vendor/product settings
parse_lite_subsystems_config.py → Lite subsystem definitions
preloader_process_data.py → Merge and validate all data
```

### 2. **Configuration Builder Pattern**
`preloader_process_data.py` acts as a builder, assembling configuration from multiple sources:
- Product config (features, subsystems)
- Vendor config (hardware settings)
- Subsystem config (component definitions)
- Platform config (board-specific settings)

### 3. **Validation Pattern**
Configuration validation before output generation:
- Check required fields present
- Validate subsystem/part existence
- Verify feature dependencies

## Data & Control Flow

### Preloader Pipeline Flow

```
1. Service Invocation
   └─> services/preloader.py calls preloader utilities

2. Configuration Parsing
   ├─> parse_vendor_product_config.py: Parse vendor/{vendor}/{product}/config.json
   ├─> parse_lite_subsystems_config.py: Parse lite subsystem definitions
   └─> Load build/subsystem_config.json

3. Data Processing
   └─> preloader_process_data.py:
       ├─> Merge product + vendor configs
       ├─> Resolve subsystem dependencies
       ├─> Process feature flags
       └─> Validate configuration

4. Output Generation
   └─> Write build_config.json to out/{product}/build_configs/
```

## Integration Points

### Key Files

| File | Purpose | Output |
|------|---------|--------|
| `parse_vendor_product_config.py` | Parse product config | Product settings dict |
| `parse_lite_subsystems_config.py` | Parse lite subsystems | Subsystem definitions |
| `preloader_process_data.py` | Merge and validate | build_config.json |

### Upstream Dependencies
- **resources/config.py**: Global configuration (product, variant)
- **resources/global_var.py**: Path constants
- **util/log_util.py**: Logging
- **util/io_util.py**: File operations

### Downstream Consumers
- **services/preloader.py**: Primary consumer
- **util/loader/**: Uses generated build_config.json
- **services/loader.py**: Reads build_config.json

### Input Files
- `vendor/{vendor}/{product}/config.json`: Product configuration
- `vendor/{vendor}/{product}/subsystem_config.json`: Product subsystems
- `build/subsystem_config.json`: Global subsystem definitions
- `build/lite/components/*.json`: Lite component definitions

### Output Files
- `out/{product}/build_configs/build_config.json`: Generated build configuration

### build_config.json Structure
```json
{
  "product_name": "rk3568",
  "device_company": "rockchip",
  "target_cpu": "arm",
  "subsystems": [
    {"subsystem": "aafwk", "components": [...]}
  ],
  "features": {...}
}
```

## Key Technical Details

### Configuration Precedence
1. Product config (highest priority)
2. Vendor config
3. Platform config
4. Default subsystem config (lowest priority)

### Dependency Resolution
- Parse component deps from ohos.build files
- Topological sort for build order
- Feature flag evaluation

See also:
- [../loader/](../loader/) - Next stage in pipeline
- [../../services/preloader.py](../../services/preloader.py) - Service wrapper
- [../../resources/config.py](../../resources/config.py) - Global configuration