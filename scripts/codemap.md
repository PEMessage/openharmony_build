# OpenHarmony Build System - Scripts Directory Codemap

## Overview

The `scripts/` directory contains Python-based build automation tools for the OpenHarmony build system. These scripts are invoked by GN (Generate Ninja) build rules to perform various build tasks including compilation, packaging, signing, and code generation.

## Directory Structure

```
scripts/
├── __init__.py                      # Package initialization
├── entry.py                         # Main build entry point
├── build_target_handler.py          # Build target processing
├── ninja_rules_parser.py            # Ninja build file parser
│
## Application/HAP Building
├── hapbuilder.py                    # HAP (HarmonyOS Ability Package) builder
├── compile_app.py                   # Application compilation with hvigor
├── compile_resources.py             # Resource compilation with restool
├── app_sign.py                      # Application signing
├── build_js_assets.py               # JavaScript asset building
├── generate_js_bytecode.py          # JavaScript bytecode generation
├── ohos_abc.py                      # Ark Bytecode (ABC) generation
│
## SDK Generation
├── gen_sdk_build_file.py            # SDK build file generation
├── sign_sdk.py                      # SDK signing for macOS
├── sign_ohos_sdk.py                 # OpenHarmony SDK signing
├── download_sdk.py                  # SDK download from CI
├── interface_mgr.py                 # SDK interface management
│
## Code Generation
├── idl.py                           # IDL (Interface Definition Language) compiler
├── cargo2gn.py                      # Rust Cargo to GN converter
├── bpf.py                           # BPF (Berkeley Packet Filter) compilation
│
## Testing & Quality
├── tools_checker.py                 # Build environment checker
├── ninja2trace.py                   # Build trace generation
├── summary_ccache_hitrate.py        # CCache statistics
├── gen_subsystem_ebpf_testcase_config.py    # eBPF test case config
├── gen_summary_ebpf_testcase_config.py      # eBPF summary config
├── generate_test_filter_info.py     # Test filter generation
├── get_warnings.py                  # Warning extraction
│
## Utilities & Helpers
├── copy_ex.py                       # Extended file copy
├── find.py                          # File finder
├── get_all_files.py                 # File enumeration
├── check_file_exist.py              # File existence checker
├── dir_exists.py                    # Directory existence checker
├── is_substring.py                  # String matching utility
├── run_shell_cmd.py                 # Shell command runner
├── run_objcopy.py                   # objcopy wrapper
├── run_objcopy_pc_mac.py            # macOS objcopy wrapper
│
## Platform Checks
├── check_linux_cpu.py               # Linux CPU detection
├── check_mac_system_and_cpu.py      # macOS system/CPU check
├── check_hvigor_hap.py              # Hvigor HAP validation
│
## Release & Compliance
├── code_release.py                  # Open source code release
├── merge_notice.py                  # NOTICE file merging
├── merge_profile.py                 # Profile merging
├── collect_publicity.py             # Publicity collection
│
## Specialized Handlers
├── kernel_permission_handler.py     # Kernel permission injection
├── asan_backup.py                   # ASan backup handling
│
## Utility Subdirectory
└── util/
    ├── __init__.py                  # Package initialization
    ├── build_utils.py               # Core build utilities
    ├── file_utils.py                # File I/O utilities
    ├── md5_check.py                 # MD5-based change detection
    ├── pycache.py                   # Python cache utilities
    ├── pyd.py                       # Python cache daemon
    ├── detect_cpu_count.py          # CPU count detection
    └── zip_and_md5.py               # ZIP and MD5 utilities
```

## Core Build Automation Scripts

### entry.py
**Purpose**: Main entry point for the OpenHarmony build system.

**Usage**: Invoked by the top-level build process to orchestrate the entire build.

**Key Functions**:
- Parses command-line arguments for product name, target CPU, build targets
- Handles SDK builds (`--product-name=ohos-sdk`)
- Supports sparse images, verbose mode, fast rebuild
- Integrates with `build.py` in the source root

**Integration**: Called by the main build orchestration (e.g., `hb build` or direct invocation).

---

### build_target_handler.py
**Purpose**: Processes and validates build targets for different platforms.

**Usage**: Translates high-level build targets to platform-specific targets.

**Key Functions**:
- Reads `parts_variants.json` for target platform variants
- Supports platform-specific builds (phone, etc.)
- Generates phony target names for ninja

**Integration**: Called during build setup to resolve target names.

---

### ninja_rules_parser.py
**Purpose**: Parses and updates Ninja build rules for multi-platform support.

**Usage**: Generates platform-specific build configurations.

**Key Functions**:
- Parses `toolchain.ninja` files
- Generates phony targets for different platforms
- Updates main `build.ninja` with subninja includes

**Integration**: Part of the GN-to-Ninja build generation pipeline.

---

## Application Building Scripts

### hapbuilder.py
**Purpose**: Builds and signs HAP (HarmonyOS Ability Package) files.

**Usage**: `python3 hapbuilder.py --hap-path <output> --hap-profile <config> ...`

**Key Functions**:
- Packages resources, assets, and native libraries
- Signs HAP files using Java hapsigner
- Supports both legacy and Stage model (app_profile mode)
- Integrates with Ark compiler for bytecode generation

**Integration**: Called by GN rules for HAP target generation.

---

### compile_app.py
**Purpose**: Compiles applications using hvigor build system.

**Usage**: `python3 compile_app.py --nodejs <path> --cwd <project_dir> --sdk-home <sdk> ...`

**Key Functions**:
- Sets up Node.js and OHPM (OpenHarmony Package Manager) environment
- Runs `ohpm install` for dependencies
- Invokes hvigorw for actual compilation
- Generates unsigned HAP path information
- Supports test HAP building
- Handles system library dependencies

**Integration**: Primary script for building 3rd-party OpenHarmony applications.

---

### compile_resources.py
**Purpose**: Compiles application resources using restool.

**Usage**: `python3 compile_resources.py --resources-dir <dir> --restool-path <tool> ...`

**Key Functions**:
- Compiles resource files (XML, images, etc.)
- Generates `ResourceTable.h` header file
- Creates packaged resources ZIP
- Extracts package name from profile

**Integration**: Called during HAP build process for resource compilation.

---

### app_sign.py
**Purpose**: Signs HAP/HSP applications.

**Usage**: `python3 app_sign.py --hapsigner <jar> --keystoreFile <path> --inFile <hap> --outFile <signed> ...`

**Key Functions**:
- Signs individual HAP files
- Batch signs multiple HAPs from JSON list
- Supports signature algorithm selection
- Handles compatible versioning

**Integration**: Final step in HAP build process.

---

### generate_js_bytecode.py
**Purpose**: Generates JavaScript bytecode using es2abc compiler.

**Usage**: `python3 generate_js_bytecode.py --src-js <file> --dst-file <out> --frontend-tool-path <path> ...`

**Key Functions**:
- Compiles JavaScript/TypeScript to Ark Bytecode (ABC)
- Supports debug info generation
- Handles module and CommonJS formats
- Supports merge-abc for incremental builds
- Patch generation for hot updates

**Integration**: Called during JS/TS asset compilation.

---

### ohos_abc.py
**Purpose**: Wrapper for Ark Bytecode generation.

**Usage**: Similar to `generate_js_bytecode.py` with GN integration.

**Key Functions**:
- GN-friendly interface for es2abc
- Supports merge mode
- Handles module type specification

**Integration**: GN action script for ABC generation.

---

## SDK Generation Scripts

### gen_sdk_build_file.py
**Purpose**: Generates BUILD.gn files for SDK modules.

**Usage**: `python3 gen_sdk_build_file.py --input-file <json> --sdk-out-dir <dir> ...`

**Key Functions**:
- Processes SDK module descriptions
- Generates prebuilt library templates
- Supports shared libraries, JARs, and Maple formats
- Handles header file installation
- Generates interface signature files

**Integration**: Part of SDK generation pipeline.

---

### interface_mgr.py
**Purpose**: Manages SDK interface compatibility checking.

**Usage**: `python3 interface_mgr.py --generate --sdk-base-dir <dir> --check_file_dir <dir>`

**Key Functions**:
- Generates SHA256 signatures for header files
- Validates SDK interface compatibility
- Creates check files for interface verification

**Integration**: Used in SDK generation to ensure API compatibility.

---

### sign_sdk.py
**Purpose**: Signs SDK binaries for macOS distribution.

**Usage**: `python3 sign_sdk.py --sdk-out-dir <dir>`

**Key Functions**:
- Codesigns binaries with Apple certificates
- Handles notarization for macOS
- Signs specific tool binaries (lldb, hdc, etc.)

**Integration**: Final step in macOS SDK packaging.

---

### download_sdk.py
**Purpose**: Downloads prebuilt SDK from CI server.

**Usage**: `python3 download_sdk.py --branch <name> --product-name <name> --api-version <ver>`

**Key Functions**:
- Fetches daily build information from CI API
- Downloads and extracts SDK archives
- Handles OHOS SDK full packages

**Integration**: Used for fetching prebuilt SDKs during setup.

---

## Code Generation Scripts

### idl.py
**Purpose**: Compiles Interface Definition Language (IDL) files.

**Usage**: `python3 idl.py --idl-path <tool> --output-archive-path <out> ...`

**Key Functions**:
- Generates C++, TypeScript, or Rust stubs from IDL
- Supports multiple output languages
- Handles dependency tracking

**Integration**: Called for IPC interface generation.

---

### cargo2gn.py
**Purpose**: Converts Rust Cargo projects to GN build files.

**Usage**: `python3 cargo2gn.py --run [--cargo-bin <path>] [--features <list>]`

**Key Functions**:
- Parses cargo build output
- Generates BUILD.gn files for Rust crates
- Handles dependencies and features
- Supports build.rs scripts
- Merges test crates

**Integration**: Used when integrating Rust third-party crates.

---

### bpf.py
**Purpose**: Compiles BPF (Berkeley Packet Filter) programs.

**Usage**: `python3 bpf.py --clang-path <path> --input-file <c_file> --output-file <o_file> ...`

**Key Functions**:
- Compiles C to BPF bytecode
- Sets BPF target architecture
- Handles include directories and defines

**Integration**: Called for eBPF program compilation.

---

## Utility Scripts

### util/build_utils.py
**Purpose**: Core utility functions for build scripts.

**Key Functions**:
- `check_output()`: Execute commands with error handling
- `temp_dir()`: Context manager for temporary directories
- `write_json()`, `read_build_vars()`: File I/O utilities
- `parse_gn_list()`: Parse GN list format
- `call_and_write_depfile_if_stale()`: Incremental build support
- `zip_dir()`, `extract_all()`, `merge_zips()`: ZIP utilities
- `add_depfile_option()`, `write_depfile()`: Dependency tracking
- `expand_file_args()`: File argument expansion

**Integration**: Used by virtually all build scripts.

---

### util/file_utils.py
**Purpose**: File operation utilities.

**Key Functions**:
- `find_top()`: Locate repository root
- `read_json_file()`, `write_json_file()`: JSON I/O
- `read_file()`, `write_file()`: Text file I/O with GN formatting

---

### util/md5_check.py
**Purpose**: MD5-based incremental build detection.

**Key Functions**:
- `call_and_record_if_stale()`: Check if targets need rebuilding
- Tracks file and string input changes
- Supports pycache integration

---

### util/pyd.py
**Purpose**: Python cache daemon for distributed builds.

**Key Functions**:
- `start_server()`: Start pycache daemon
- `stop_server()`: Stop daemon
- `show_statistics()`: Cache hit/miss statistics
- `manage_cache_contents()`: Cache cleanup (40GB/15 days limit)

---

## Quality & Analysis Scripts

### tools_checker.py
**Purpose**: Validates build environment.

**Usage**: `python3 tools_checker.py`

**Key Functions**:
- Checks OS version (Ubuntu 18.04/20.04/22.04)
- Validates required packages are installed
- Uses `build_package_list.json` for package lists

---

### ninja2trace.py
**Purpose**: Converts Ninja logs to Chrome trace format.

**Usage**: `python3 ninja2trace.py --ninja-log <file> --trace-file <out> --duration-file <out>`

**Key Functions**:
- Parses `.ninja_log` files
- Generates Chrome trace viewer compatible JSON
- Calculates build duration statistics

---

### code_release.py
**Purpose**: Packages open source code for release.

**Usage**: `python3 code_release.py --output <tar.gz> --root-dir <dir> --scan-dirs <dirs> --scan-licenses <licenses>`

**Key Functions**:
- Scans for `README.OpenSource` files
- Filters by license type
- Creates tarball of releaseable code

---

## Integration with Build System

### GN Integration

Most scripts are designed to be called as `action()` targets in GN:

```gn
action("generate_hap") {
  script = "//build/scripts/hapbuilder.py"
  inputs = [ ... ]
  outputs = [ ... ]
  args = [ ... ]
}
```

### Depfile Support

Scripts support Ninja depfiles for proper incremental builds:
- Use `build_utils.add_depfile_option(parser)` to add `--depfile` argument
- Use `build_utils.call_and_write_depfile_if_stale()` for automatic depfile generation

### Common Patterns

1. **Argument Parsing**: All scripts use `argparse` or `optparse`
2. **Error Handling**: Use `build_utils.CalledProcessError` for command failures
3. **Logging**: Print progress messages to stdout
4. **Exit Codes**: Return 0 on success, non-zero on failure

## Summary

The `scripts/` directory is the core automation layer for OpenHarmony's build system, providing:

- **46 Python scripts** for various build tasks
- **8 utility modules** in `util/` subdirectory
- Support for multiple languages (C/C++, Rust, JavaScript, Java)
- Cross-platform support (Linux, macOS)
- Incremental build support via MD5 checking and depfiles
- Integration with GN/Ninja build system
- SDK generation and signing capabilities
- HAP building and signing for application distribution
