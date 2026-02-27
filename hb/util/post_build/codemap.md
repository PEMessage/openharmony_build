# util/post_build/ - Post-Build Processing

Post-build utilities for analyzing build outputs, calculating ROM statistics, and generating reports after compilation completes.

## Responsibility

**Primary Purpose:** Post-build analysis and reporting utilities.

**Key Responsibilities:**
- **ROM Statistics (`part_rom_statistics.py`)**: Calculate per-part ROM usage from build artifacts

## Design Patterns

### 1. **Analyzer Pattern**
`part_rom_statistics.py` analyzes ELF binaries and map files to calculate code/data sizes per component.

### 2. **Reporter Pattern**
Generate structured reports in JSON/text formats for build metrics.

## Data & Control Flow

### Post-Build Analysis Flow

```
1. Build Completion
   └─> Ninja build finishes

2. Statistics Collection
   └─> part_rom_statistics.py analyzes:
       ├─> ELF binary sizes
       ├─> Map file analysis
       └─> Per-part breakdown

3. Report Generation
   └─> Output JSON/text reports to out/{product}/images/
```

## Integration Points

### Key Files

| File | Purpose | Output |
|------|---------|--------|
| `part_rom_statistics.py` | ROM size analysis | ROM statistics reports |

### Upstream Dependencies
- **resources/config.py**: Get product/build info
- **util/log_util.py**: Logging

### Downstream Consumers
- **services/loader.py**: May invoke post-build analysis
- **CI/CD systems**: Consume generated reports

### Input Files
- `out/{product}/images/*.elf`: Compiled binaries
- `out/{product}/images/*.map`: Linker map files

### Output Files
- `out/{product}/images/rom_statistics.json`: Per-part size breakdown

## Key Technical Details

### Size Calculation
Analyzes ELF sections:
```python
# .text, .rodata, .data, .bss sections
# Per-part attribution via symbol tables
```

### Report Format
```json
{
  "part_name": {
    "text_size": 1234,
    "rodata_size": 567,
    "data_size": 89,
    "bss_size": 0
  }
}
```

See also:
- [../loader/](../loader/) - Build loading stage
- [../preloader/](../preloader/) - Pre-build configuration stage