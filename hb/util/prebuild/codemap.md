# util/prebuild/ - Pre-Build Setup

Pre-build utilities for managing prebuilt binaries, applying patches, and preparing the build environment before compilation.

## Responsibility

**Primary Purpose:** Pre-build environment preparation and dependency management.

**Key Responsibilities:**
- **Patch Processing (`patch_process.py`)**: Apply patches to source code or prebuilt components

## Design Patterns

### 1. **Patch Application Pattern**
`patch_process.py` manages patching of source code:
- Apply `.patch` files to source directories
- Track applied patches to avoid double-application
- Handle patch conflicts and failures

## Data & Control Flow

### Pre-Build Flow

```
1. Build Initialization
   └─> services/preloader.py or prebuild phase starts

2. Patch Application (if needed)
   └─> patch_process.py:
       ├─> Scan for patch files
       ├─> Validate patch applicability
       └─> Apply patches to source tree

3. Environment Ready
   └─> Continue to preloader/loader stages
```

## Integration Points

### Key Files

| File | Purpose | Input/Output |
|------|---------|--------------|
| `patch_process.py` | Patch management | .patch files → modified source |

### Upstream Dependencies
- **resources/global_var.py**: Path constants
- **util/log_util.py**: Logging
- **util/io_util.py**: File operations

### Downstream Consumers
- **services/preloader.py**: May invoke patching
- **services/prebuilts.py**: Download and patch prebuilts

### Input Files
- `build/patches/*.patch`: Patch files
- `.applied_patches`: Tracking file

## Key Technical Details

### Patch Application
```python
# Check if patch already applied
# Validate patch can apply cleanly
# Apply using git apply or patch command
# Record in tracking file
```

See also:
- [../preloader/](../preloader/) - Configuration generation
- [../../services/prebuilts.py](../../services/prebuilts.py) - Prebuilt management