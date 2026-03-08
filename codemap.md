# Repository Atlas: OpenHarmony Build System

## Project Responsibility

This is the **OpenHarmony Build System** - a comprehensive GN/Ninja-based build framework that compiles OpenHarmony source code into system images, SDKs, and application packages. It provides the `hb` CLI tool for build orchestration and supports both standard (rich) and lite (IoT/embedded) build targets.

## System Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                        User Interface                            │
│                     hb set / hb build / hb clean                 │
└─────────────────────────────────────────────────────────────────┘
                                │
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Build Orchestration                         │
│    hb/ → main.py → Modules → Services (Preloader/Loader/GN/Ninja)│
└─────────────────────────────────────────────────────────────────┘
                                │
              ┌─────────────────┼─────────────────┐
              ▼                 ▼                 ▼
┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐
│   core/gn/      │  │     ohos/       │  │     lite/       │
│  BUILD.gn       │  │  Templates &    │  │  Lite Build     │
│  Entry Point    │  │  Packaging      │  │  System         │
└─────────────────┘  └─────────────────┘  └─────────────────┘
              │                 │                 │
              └─────────────────┼─────────────────┘
                                ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Build Execution                             │
│           GN → Ninja → Compilers → Images/Packages               │
└─────────────────────────────────────────────────────────────────┘
```

## System Entry Points

| File | Purpose |
|------|---------|
| `hb/__main__.py` | CLI entry point for `hb` command |
| `hb/main.py` | Main build orchestrator, workspace validation |
| `core/gn/BUILD.gn` | Primary GN build targets definition |
| `core/gn/dotfile.gn` | GN dotfile configuration |
| `ohos.gni` | Aggregated GNI imports for modules |
| `ohos_var.gni` | Global build variable definitions |
| `gn_helpers.py` | GN helper functions for Python scripts |

## Directory Map (Aggregated)

| Directory | Responsibility Summary | Detailed Map |
|-----------|------------------------|--------------|
| `hb/` | **OpenHarmony Build CLI Tool** - Python-based command-line interface with 11 command modules (build, set, clean, env, tool, etc.), argument resolvers, and build services (Preloader, Loader, GN, Ninja, HPM). Implements Facade, Factory, and Strategy patterns. | [View Map](hb/codemap.md) |
| `ohos/` | **Core Build Templates** - GN templates for OpenHarmony component building, packaging, and image generation. Contains 17+ subdirs: kits, ndk, sdk, app, sa_profile, notice, images, packages, sbom, etc. Defines how parts aggregate modules into system images. | [View Map](ohos/codemap.md) |
| `common/` | **Runtime Libraries** - Shared runtime components (libc++, musl, sanitizers) required on all OpenHarmony images. Includes ASan, HWASan, TSan, UBSan configurations with multi-arch support (arm, arm64, x86_64, riscv64). | [View Map](common/codemap.md) |
| `core/` | **GN Entry Point** - Central configuration hub bridging `hb` tool with GN/Ninja. Contains main BUILD.gn with build targets, dotfile.gn for GN config, and security allowlist (200+ entries) for exec_script(). | [View Map](core/codemap.md) |
| `lite/` | **Lite Build System** - Simplified build for IoT/embedded devices running LiteOS/UniProton. Provides lite_subsystem, lite_component, lite_library templates and board-configurable toolchains (GCC, Clang, IAR). | [View Map](lite/codemap.md) |
| `tools/` | **Build Utilities** - 30+ Python scripts for dependency analysis, component management, product config migration, and static code checking. Includes csct.py for BUILD.gn validation and module_deps_tree.py for visualization. | [View Map](tools/codemap.md) |
| `scripts/` | **Build Automation** - 46 Python scripts serving as GN/Ninja action handlers. Categories: entry points, HAP building, SDK generation, code generation (idl, cargo2gn, bpf), and utility infrastructure in `util/`. | [View Map](scripts/codemap.md) |
| `config/` | **Configuration Files** - JSON configuration files for build standards, component whitelists, subsystem configs, and prebuilts configuration. | [View Map](config/codemap.md) |
| `templates/` | **Build Templates** - GN template definitions for test frameworks, IDL, Rust, and other specialized build scenarios. | [View Map](templates/codemap.md) |
| `misc/` | **Miscellaneous Utilities** - Additional build support utilities and scripts. | [View Map](misc/codemap.md) |
| `rust/` | **Rust Build Support** - Rust-specific build configurations and cargo integration. | [View Map](rust/codemap.md) |
| `toolchain/` | **Toolchain Definitions** - Cross-compilation toolchain configurations for various target architectures. | [View Map](toolchain/codemap.md) |

## Key Configuration Files

| File | Description |
|------|-------------|
| `bundle.json` | Build system component descriptor |
| `subsystem_config.json` | Subsystem to path mappings |
| `ohos.gni` | Common GNI imports for modules |
| `ohos_var.gni` | Global build variables |
| `test.gni` | Test framework templates |
| `common.gni` | Common build definitions |
| `version.gni` | Build system version |
| `prebuilts_config.json` | Prebuilt binary configuration |
| `component_compilation_whitelist.json` | Component build allowlist |
| `compile_standard_whitelist.json` | Compiler standard allowlist |

## Build Process Flow

```
1. Product Selection (hb set)
   └── User selects product from //vendor or //product

2. Preload Phase (hb/services/preloader.py)
   ├── Parse product_config.json
   ├── Generate parts_info.json
   ├── Generate subsystem_config.json
   └── Output to out/{product}/build_configs/

3. Load Phase (hb/services/loader.py)
   ├── Process features
   ├── Generate build_vars.json
   └── Prepare toolchain configs

4. GN Phase (hb/services/gn.py → core/gn/)
   ├── gn gen with args from build_configs/
   ├── Read BUILDCONFIG.gn
   ├── Process //build/ohos/ templates
   └── Generate ninja files to out/{product}/

5. Ninja Phase (hb/services/ninja.py)
   └── ninja -C out/{product}/ <targets>

6. Post-Build Phase (hb/services/post_build.py)
   ├── Package generation
   ├── Image creation (ext4, f2fs, cpio)
   ├── SDK packaging (optional)
   └── SBOM generation (optional)
```

## Integration Points

### External Tools
- **GN** - Meta-build system generating Ninja files
- **Ninja** - Low-level build execution
- **HPM** - HarmonyOS Package Manager for independent builds
- **HDC** - HarmonyOS Device Connector for deployment
- **CCache** - Compiler cache for faster rebuilds

### Build Phases (hb/services/)
The build is organized into 12 phases:
1. `PRE_BUILD` - Environment setup
2. `PRE_LOAD` - Preloader execution
3. `LOAD` - Loader execution
4. `PRE_TARGET_GENERATION` - Target preparation
5. `TARGET_GENERATION` - Target generation
6. `GN` - GN meta-build
7. `NINJA` - Ninja build execution
8. `POST_BUILD` - Post-processing
9. `IMAGE_GENERATION` - System image creation
10. `PACKAGE_GENERATION` - Package creation
11. `SDK_GENERATION` - SDK packaging
12. `INDEP_COMPILATION` - Independent component builds

### Error Code Taxonomy
Error codes are 4-digit numbers where:
- First digit indicates stage (1=preloader, 2=loader, 3=GN, 4=ninja)
- Remaining digits indicate specific error type

## Design Patterns Used

1. **Facade Pattern** - `hb/__main__.py` hides workspace validation
2. **Factory Method Pattern** - `hb/main.py` creates command modules
3. **Dependency Injection** - Services injected via constructors
4. **Strategy Pattern** - Module selection via dictionary mapping
5. **Template Method Pattern** - `BuildModuleInterface` defines phases
6. **Singleton Pattern** - `Config` class for global state
7. **Observer Pattern** - Build phase callbacks

## Security Features

- `core/gn/ohos_exec_script_allowlist.gni` - Whitelist of 200+ paths allowed to use `exec_script()`
- Sanitizer support (ASan, HWASan, TSan, UBSan) for debugging
- Stack protector and PIE/PIC compilation flags

## Statistics

- **Total Files Tracked**: 501
- **Python Scripts**: 150+
- **GN/GNI Templates**: 100+
- **Main Directories**: 12
- **Build Targets**: Thousands (dynamically generated from product config)

## Quick Reference

| Task | Command | Entry Point |
|------|---------|-------------|
| Set product | `hb set` | `hb/modules/set.py` |
| Build | `hb build` | `hb/modules/build.py` |
| Clean | `hb clean` | `hb/modules/clean.py` |
| Check env | `hb env` | `hb/modules/env.py` |
| Independent build | `hb build --indep` | `hb/modules/indep_build.py` |
