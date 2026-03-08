# lite/ - OpenHarmony Lite Build System

## Overview

The `lite/` directory contains the **Lite Build System** for OpenHarmony, serving as the entry point for the `hb` (Harmony Build) tool and the preloader. It provides a streamlined build framework specifically designed for resource-constrained devices (IoT, embedded systems) running LiteOS or Linux kernels.

## Responsibility

The lite build system is responsible for:

1. **Build Orchestration**: Managing the complete build flow from product configuration to image generation
2. **Cross-Compilation Support**: Providing toolchain abstraction for GCC, Clang, and IAR ARM compilers
3. **Component Management**: Organizing subsystems and components through JSON/GN configuration
4. **Root Filesystem Creation**: Generating rootfs images (JFFS2, YAFFS2, VFAT, EXT4) for target devices
5. **NDK Generation**: Building Native Development Kit for third-party application development
6. **HAP Packaging**: Building and signing Harmony Ability Packages (HAPs)
7. **Test Framework Integration**: Generating test metadata and managing test resources
8. **License Compliance**: Generating NOTICE files for third-party components

## How Lite Build Differs from Standard Build

| Aspect | Lite Build | Standard Build |
|--------|------------|----------------|
| **Target Devices** | Resource-constrained IoT/embedded devices | Rich devices (phones, tablets, etc.) |
| **Kernel Support** | liteos_a, liteos_m, Linux, UniProton | Linux (standard) |
| **Build Tool** | GN + Ninja with lite-specific templates | GN + Ninja with full-featured templates |
| **Component Model** | Simplified `lite_component` template | Full `ohos_part` with complex features |
| **Toolchain** | Board-configurable (GCC/Clang/IAR) | Primarily Clang/LLVM |
| **Sysroot** | Custom musl-based sysroot | Standard musl libc |
| **Output** | Simplified rootfs images | Full system images with complex partitioning |

### Key Differences in Detail

1. **GN Build Configuration**:
   - Uses `BUILDCONFIG.gn` instead of standard `BUILD.gn` at root
   - Defines `ohos_lite = true` to distinguish from standard build
   - Simplified target type definitions (executable, static_library, shared_library, source_set)

2. **Subsystem/Component Templates**:
   - `lite_subsystem` - Defines a subsystem with component list
   - `lite_component` - Defines a component with feature list
   - `lite_library` - Unified library template handling static/shared/executable

3. **Target List Generation**:
   - `lite_target_list.gni` dynamically generates build targets from product configuration
   - Filters targets based on kernel type and userspace-only builds

## Entry Points

### For `hb` (Harmony Build Tool)

The `hb` tool uses the lite build system through these entry points:

1. **Build Entry**: `//build/lite:ohos` (defined in `BUILD.gn`)
   - Main target that builds all configured subsystems and components
   - Depends on `lite_target_list` for target enumeration

2. **Product Entry**: `//build/lite:product`
   - Builds product-specific configurations
   - References `${product_path}` from product configuration

3. **NDK Entry**: `//build/lite:ndk`
   - Builds Native Development Kit when `ohos_build_ndk = true`

### For Preloader

The preloader interacts with:

1. **Variable Definitions**: `ohos_var.gni`
   - Defines all global build variables
   - Reads product configuration from `${product_config_path}/config.json`

2. **Toolchain Setup**: `toolchain/BUILD.gn`
   - Configures compiler based on board settings
   - Supports GCC, Clang, and IAR ARM toolchains

## Directory Structure

```
lite/
├── BUILD.gn                    # Main build targets (ohos, product, ndk, prebuilts)
├── ohos_var.gni                # Global build variables and product config parsing
├── lite_target_list.gni        # Dynamic target list generation from product config
├── utils.py                    # Common Python utilities for build scripts
├──
├── config/                     # Build configuration templates and settings
│   ├── BUILDCONFIG.gn          # GN build configuration (toolchain, defaults)
│   ├── BUILD.gn                # Compiler configs (security, optimization, arch)
│   ├── component/              # Component templates
│   │   └── lite_component.gni  # lite_library, lite_component, build_ext_component templates
│   ├── subsystem/              # Subsystem templates
│   │   ├── lite_subsystem.gni  # lite_subsystem template
│   │   ├── aafwk/              # Ability framework configs
│   │   ├── graphic/            # Graphic subsystem configs
│   │   └── hiviewdfx/          # HiView DFX configs
│   ├── toolchain/              # Toolchain configuration
│   ├── kernel/                 # Kernel-specific configs
│   └── test.gni                # Test configuration
│
├── toolchain/                  # Toolchain definitions
│   ├── BUILD.gn                # Toolchain instantiation (gcc/clang/iccarm)
│   ├── clang.gni               # Clang toolchain template
│   ├── gcc.gni                 # GCC toolchain template
│   └── iccarm.gni              # IAR ARM toolchain template
│
├── components/                 # Component metadata
│   └── communication.json      # Communication subsystem component definitions
│
├── testfwk/                    # Test framework support
│   ├── gen_testfwk_info.py     # Generate test framework metadata
│   ├── gen_module_list_files.py # Generate test module lists
│   └── lite_testcase_resource_copy.py # Test resource management
│
├── ndk/                        # Native Development Kit
│   ├── ndk.gni                 # NDK templates (ndk_lib, copy_files, ndk_toolchains)
│   ├── BUILD.gn                # NDK build targets
│   ├── archive_ndk.py          # NDK packaging script
│   ├── build/                  # Standalone NDK build system
│   │   ├── build.py            # NDK build entry script
│   │   ├── BUILD.gn            # NDK build config
│   │   └── toolchain/          # NDK toolchain configs
│   └── doc/                    # NDK documentation generation
│
├── make_rootfs/                # Root filesystem image creation
│   ├── rootfsimg_linux.sh      # Linux rootfs image builder
│   ├── rootfsimg_liteos.sh     # LiteOS rootfs image builder
│   └── dmverity_linux.sh       # DM-Verity for Linux
│
└── [Build Scripts]
    ├── build_ext_components.py # External component build wrapper
    ├── hap_pack.py             # HAP packaging and signing
    ├── copy_files.py           # File/directory copy utility
    ├── run_shell_cmd.py        # Shell command wrapper
    └── gen_module_notice_file.py # Third-party license file generator
```

## Key Modules and Responsibilities

### 1. Build Configuration (`config/`)

#### `BUILDCONFIG.gn`
- **Purpose**: Core GN build configuration
- **Key Functions**:
  - Sets target OS and CPU based on board configuration
  - Configures toolchain (GCC/Clang/IAR ARM)
  - Sets default compiler flags and configs
  - Defines target defaults for executables, libraries, and source sets
  - Handles ccache/xcache integration

#### `BUILD.gn` (config/)
- **Purpose**: Compiler and linker configuration definitions
- **Key Configs**:
  - `cpu_arch`: Architecture-specific flags
  - `kernel_macros`: Kernel type defines (__LITEOS__, __LINUX__, etc.)
  - `security`: Security hardening flags (stack protector, RELRO, NX stack)
  - `common`: Common compiler flags (-Wall, -fno-common, etc.)
  - `ohos_clang`: Clang-specific settings with LLD linker
  - `board_config`: Board-specific flags and include paths

#### `lite_component.gni`
- **Purpose**: Core templates for lite build system
- **Templates**:
  - `lite_library`: Unified library template handling static, shared, and executable targets
  - `lite_component`: Component definition with feature list
  - `build_ext_component`: Wrapper for external build systems
  - `ohos_tools`: Tools target with host-specific configs
  - `generate_notice_file`: License file generation for third-party code

#### `lite_subsystem.gni`
- **Purpose**: Subsystem organization templates
- **Templates**:
  - `lite_subsystem`: Groups components into subsystems
  - `lite_subsystem_test`: Test variant of subsystem
  - `lite_subsystem_sdk`: SDK generation for subsystems
  - `lite_vendor_sdk`: Vendor-specific SDK generation

### 2. Toolchain Management (`toolchain/`)

#### `clang.gni` / `gcc.gni` / `iccarm.gni`
- **Purpose**: Toolchain definition templates
- **Key Tools Defined**:
  - `cc`: C compiler
  - `cxx`: C++ compiler
  - `asm`: Assembler
  - `alink`: Static library archiver
  - `solink`: Shared library linker
  - `link`: Executable linker
  - `stamp`: Timestamp file creation
  - `copy`: File copying

#### `BUILD.gn` (toolchain/)
- Instantiates the appropriate toolchain based on `board_toolchain_type`
- Sets compiler commands from board configuration

### 3. Target List Generation (`lite_target_list.gni`)

- **Purpose**: Dynamically generates build target list from product configuration
- **Flow**:
  1. Reads `${product_config_path}/config.json`
  2. Reads `parts_modules_info.json` for module mapping
  3. For each subsystem in product config:
     - Reads mini_adapter JSON for subsystem parts
     - Validates components exist and support current kernel
     - Adds component module lists to `lite_target_list`
  4. Adds device/product targets (except for liteos_m kernel)

### 4. Variable Definitions (`ohos_var.gni`)

- **Purpose**: Global build variables
- **Key Variables**:
  - `ohos_version`: OpenHarmony version string
  - `product`, `device_path`, `product_path`: Product/device paths
  - `ohos_build_type`: debug/release
  - `ohos_kernel_type`: liteos_a, liteos_m, linux, uniproton
  - `ohos_build_*_command`: Current toolchain commands
  - `ohos_current_sysroot`: Sysroot path for cross-compilation
  - `ohos_lite`: Set to `true` to identify lite build

### 5. NDK System (`ndk/`)

#### `ndk.gni`
- **Templates**:
  - `ndk_lib`: Copy library and headers to NDK output
  - `copy_files`: Generic file/directory copying
  - `ndk_toolchains`: Copy toolchain binaries

#### `BUILD.gn` (ndk/)
- Copies compiler (GCC or Clang based on configuration)
- Copies build scripts, samples, and sysroot
- Collects all NDK libraries from various subsystems
- Creates final NDK zip archive

#### `build/build.py`
- Standalone build script for NDK users
- Provides `build` and `clean` commands
- Uses prebuilt GN and Ninja from NDK

### 6. Root Filesystem (`make_rootfs/`)

#### `rootfsimg_linux.sh` / `rootfsimg_liteos.sh`
- Creates root filesystem images for target devices
- Supported filesystems:
  - **JFFS2**: Journalling Flash File System v2
  - **YAFFS2**: Yet Another Flash File System v2 (LiteOS only)
  - **VFAT**: FAT32 for SD cards
  - **EXT4**: Extended filesystem v4 (Linux only)
- Handles device file permissions and ownership

### 7. HAP Packaging (`hap_pack.py` / `hap_pack.gni`)

#### `hap_pack.gni`
- Template: `hap_pack` - Packages and signs HAP files
- Supports both local signing and server-based signing
- Configures signing algorithms and certificates

#### `hap_pack.py`
- Python script for HAP packaging workflow:
  1. Package resources with `app_packing_tool.jar`
  2. Sign package with `hap-sign-tool.jar`
- Supports remote signing via environment variables

### 8. Test Framework (`testfwk/`)

#### `gen_testfwk_info.py`
- Generates test framework metadata JSON
- Maps subsystems to components for test execution

#### `gen_module_list_files.py`
- Creates module list files for test discovery
- Generates `.sources` files for test case resources

#### `lite_testcase_resource_copy.py`
- Copies test resources based on XML configuration
- Supports both resource directory and build output paths

### 9. Utility Scripts

#### `utils.py`
- Common utilities for Python build scripts:
  - `exec_command()`: Execute shell commands with logging
  - `check_output()`: Capture command output
  - `read_json_file()`: JSON file reader
  - `makedirs()`: Directory creation
  - `CallbackDict`: Event callback system

#### `build_ext_components.py`
- Wrapper for building external components
- Handles prebuild steps and command execution
- Captures timing and error logs

#### `copy_files.py`
- File and directory copy utility
- Handles symlinks and git/repo exclusion

#### `gen_module_notice_file.py`
- Generates license NOTICE files for third-party components
- Reads `README.OpenSource` and `COPYRIGHT.OpenSource` files
- Creates formatted license files in output

## Integration with Broader Build System

### 1. Product Configuration Integration

```
Product config (config.json)
    ↓
ohos_var.gni (parses JSON)
    ↓
lite_target_list.gni (generates target list)
    ↓
BUILD.gn:ohos target (builds all targets)
```

### 2. Toolchain Integration

```
Board config (config.gni)
    ↓
BUILDCONFIG.gn (reads board_toolchain_*)
    ↓
toolchain/BUILD.gn (instantiates toolchain)
    ↓
Compiler commands set in ohos_current_*_command
```

### 3. Subsystem/Component Integration

```
Subsystem definition (lite_subsystem.gni)
    ↓
Component definition (lite_component.gni)
    ↓
Library targets (lite_library template)
    ↓
Actual build targets (executable, static_library, shared_library)
```

### 4. External Build System Integration

```
External component BUILD.gn
    ↓
build_ext_component template
    ↓
build_ext_components.py
    ↓
External build command (make, cmake, etc.)
```

### 5. NDK Integration

```
Native API libraries (various subsystems)
    ↓
ndk_lib template
    ↓
ndk/BUILD.gn:ndk_build
    ↓
ndk/BUILD.gn:ndk (archive action)
    ↓
archive_ndk.py
    ↓
NDK zip package
```

## Build Flow Summary

1. **Configuration Phase**:
   - `hb set` selects product → writes product config path
   - GN reads `ohos_var.gni` → parses product `config.json`
   - `lite_target_list.gni` generates target list from product subsystems

2. **Generation Phase**:
   - GN generates build.ninja from all BUILD.gn files
   - Toolchain is configured based on board settings
   - Default configs applied to all targets

3. **Build Phase**:
   - Ninja executes build.ninja
   - `//build/lite:ohos` builds all lite targets
   - `//build/lite:product` builds product-specific code
   - External components built via `build_ext_components.py`

4. **Packaging Phase** (optional):
   - `make_rootfs` scripts create filesystem images
   - `hap_pack.py` packages and signs HAPs
   - `archive_ndk.py` creates NDK distribution

5. **Output**:
   - Compiled binaries in `$root_out_dir/`
   - Libraries in `$root_out_dir/libs/`
   - Rootfs images for flashing
   - NDK package for developers

## Security and Compliance

### Security Features
- Stack protector (`-fstack-protector-all`)
- Position Independent Executable (PIE)
- Position Independent Code (PIC) for shared libraries
- RELRO (Relocation Read-Only)
- NX stack (`-z noexecstack`)
- Immediate binding (`-z now`)

### License Compliance
- `gen_module_notice_file.py` tracks third-party licenses
- `README.OpenSource` files required for third_party components
- NOTICE files generated for each output binary
