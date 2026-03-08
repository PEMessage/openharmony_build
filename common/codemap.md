# OpenHarmony Build System - Common Directory Codemap

## Overview

The `common/` directory contains shared runtime libraries, sanitizer configurations, and essential system components that are installed across all OpenHarmony images (system, vendor, ramdisk, updater). These are prebuilt binaries and configuration files required for the runtime environment.

## Directory Structure

```
common/
├── BUILD.gn                  # Main build file - defines common_packages group
├── libcpp/                   # C++ standard library (libc++)
│   └── BUILD.gn             # libc++_shared.so installation rules
├── musl/                     # musl libc runtime
│   ├── BUILD.gn             # musl linker and libc installation
│   ├── ld-musl-arm.path     # ARM library search paths
│   ├── ld-musl-aarch64.path # ARM64 library search paths
│   └── ld-musl-riscv64.path # RISC-V library search paths
├── asan/                     # AddressSanitizer (ASan) runtime
│   ├── BUILD.gn             # ASan/HWASan/TSan library installation
│   ├── asan.options         # ASan runtime options
│   ├── asan.cfg             # Init system configuration for ASan
│   ├── tsan.options         # ThreadSanitizer options
│   └── build_mixed_asan.sh  # Script for building mixed ASan images
└── ubsan/                    # UndefinedBehaviorSanitizer runtime
    ├── BUILD.gn             # UBSan library installation
    └── ubsan.cfg            # UBSan init configuration
```

## 1. Main Build File (BUILD.gn)

**Location**: `common/BUILD.gn`

**Purpose**: Defines the `common_packages` group that aggregates all common runtime dependencies.

**Key Components**:
- **musl_install**: musl libc linker and libraries
- **libcpp_install**: C++ standard library (libc++_shared.so)
- **ASan libraries**: libclang_rt.asan.so (all architectures), libclang_rt.hwasan.so, libclang_rt.tsan.so (arm64 only)
- **ASan/TSan configs**: asan.options, asan.cfg, tsan.options (when sanitizers enabled)
- **UBSan libraries**: libclang_rt.ubsan_minimal.so, libclang_rt.ubsan_standalone.so, ubsan.cfg (when no sanitizers)

**Conditional Logic**:
- HWASan and TSan libraries only included for `target_cpu == "arm64"`
- Sanitizer configs conditionally included based on `is_asan`, `is_tsan` flags
- UBSan used as fallback when ASan/TSan are not enabled

## 2. C++ Standard Library (libcpp/)

**Location**: `common/libcpp/BUILD.gn`

**Purpose**: Installs the libc++ shared library for C++ runtime support.

**Targets**:
- `libcpp_install`: Group target for installation
- `libc++_shared.so`: Prebuilt shared library from Clang toolchain
  - Supports arm, arm64, x86_64 architectures
  - HWASan variant for arm64 when `use_hwasan == true`
  - Installs to `system` image
  - Strips debug symbols, generates mini-debug info

**Source Paths**:
- ARM: `${clang_stl_path}/arm-linux-ohos/libc++_shared.so`
- ARM64: `${clang_stl_path}/aarch64-linux-ohos/libc++_shared.so`
- ARM64+HWASan: `${clang_stl_path}/aarch64-linux-ohos/hwasan/libc++_shared.so`
- x86_64: `${clang_stl_path}/x86_64-linux-ohos/libc++_shared.so`

## 3. musl libc Runtime (musl/)

**Location**: `common/musl/`

**Purpose**: Provides the musl libc linker and C++ runtime libraries for OpenHarmony.

### 3.1 BUILD.gn Targets

**Main Group**: `musl_install`
Dependencies:
- `musl-libcxx.so` / `musl-libcxx.so_arm64e`: C++ standard library for musl
- `musl_ld_path_etc_cfg`: Library search path configuration
- `//third_party/musl:musl_libs`: Core musl libraries
- Architecture-specific linker libraries

**Linker Libraries**:
- `ld-musl-riscv64.so.1`: RISC-V 64-bit dynamic linker
- `ld-musl-arm.so.1`: ARM/ARM64 dynamic linker
- `ld-musl-arm.so.1_arm64e`: ARM64E variant for Apple Silicon-like extensions

**Configuration**:
- `musl_ld_path_etc_cfg`: Installs library search path files to `etc/`

### 3.2 Library Search Path Files

These files define the dynamic linker library search paths:

**ld-musl-arm.path** (ARM 32-bit):
```
/system/lib:/vendor/lib:/vendor/lib/chipsetsdk:/system/lib/ndk:...
```

**ld-musl-aarch64.path** (ARM 64-bit):
```
/system/lib64:/vendor/lib64:/system/lib:/vendor/lib:...
```

**ld-musl-riscv64.path** (RISC-V 64-bit):
```
/system/lib64:/vendor/lib64:/system/lib:/vendor/lib:...
```

Paths include SDK-specific directories (chipsetsdk, platformsdk, ndk, etc.)

## 4. AddressSanitizer Runtime (asan/)

**Location**: `common/asan/`

**Purpose**: Provides AddressSanitizer (ASan), Hardware-assisted AddressSanitizer (HWASan), and ThreadSanitizer (TSan) runtime libraries and configurations.

### 4.1 BUILD.gn Targets

**Shared Libraries**:
- `libclang_rt.asan.so`: AddressSanitizer runtime (all architectures)
- `libclang_rt.hwasan.so`: Hardware-assisted ASan (arm64 only)
- `libclang_rt.tsan.so`: ThreadSanitizer runtime (arm64 only)

**Configuration Files**:
- `asan.options`: Installed to `system/etc/asan.options`
- `asan.cfg`: Init system config for setting up ASan environment
- `tsan.options`: ThreadSanitizer runtime options

**Installation Images**:
- ASan: system, ramdisk, updater
- HWASan: system, updater (+ ramdisk when `use_hwasan`)
- TSan: system, ramdisk, updater

### 4.2 asan.options

Runtime options for AddressSanitizer:
```
quarantine_size_mb=256
max_redzone=2048
thread_local_quarantine_size_kb=256
detect_odr_violation=0
allocator_may_return_null=1
abort_on_error=0
halt_on_error=1
print_module_map=1
memory_debug=1
heap_history_size_main_thread=1023000
...
```

### 4.3 asan.cfg

Init system configuration for ASan/HWASan/TSan setup:
```json
{
    "jobs": [{
        "name": "pre-init",
        "cmds": [
            "setrlimit RLIMIT_STACK unlimited unlimited",
            "export ASAN_OPTIONS log_path=/dev/asan/asan.log:include=/system/etc/asan.options",
            "export HWASAN_OPTIONS log_path=/dev/hwasan/hwasan.log:include=/system/etc/asan.options",
            "export TSAN_OPTIONS include=/system/etc/tsan.options"
        ]
    }, {
        "name": "early-fs",
        "cmds": [
            "mkdir /data/log/sanitizer/asan/ ...",
            "mkdir /dev/asan/ ...",
            "mount none /data/log/sanitizer/asan /dev/asan bind"
        ]
    }]
}
```

### 4.4 tsan.options

ThreadSanitizer runtime options:
```
allow_addr2line=1
allocator_may_return_null=1
detect_deadlocks=1
second_deadlock_stack=1
history_size=7
print_full_thread_history=1
...
```

### 4.5 build_mixed_asan.sh

**Purpose**: Script for building "mixed" ASan images where some services run with ASan instrumentation.

**Features**:
- Builds both ASan and non-ASan versions
- Creates system/vendor images with selective ASan enablement
- Supports custom service configuration groups
- Can place ASan binaries in `/data/asan` or in-place

**Usage**:
```bash
./build_mixed_asan.sh [options] --gn-args ...
```

**Key Operations**:
1. Build ASan variant → `out.a/`
2. Build non-ASan variant → `out/`
3. Merge images with selective ASan enablement
4. Modify init configs to run specific services with ASan

## 5. UndefinedBehaviorSanitizer Runtime (ubsan/)

**Location**: `common/ubsan/`

**Purpose**: Provides UBSan runtime libraries for detecting undefined behavior (used when ASan/TSan are not enabled).

### 5.1 BUILD.gn Targets

**Group**: `ubsan`
Includes:
- `libclang_rt.ubsan_standalone.so`: Full UBSan runtime
- `libclang_rt.ubsan_minimal.so`: Minimal UBSan runtime
- `ubsan.cfg`: Init system configuration

**Installation**:
- Images: system, updater
- Inner API tags: platformsdk, chipsetsdk

### 5.2 ubsan.cfg

UBSan init configuration:
```json
{
    "jobs": [{
        "name": "pre-init",
        "cmds": [
            "export UBSAN_OPTIONS print_stacktrace=1:print_module_map=2:log_exe_name=1"
        ]
    }, {
        "name": "post-fs-data",
        "cmds": [
            "mkdir /data/log/sanitizer/ubsan/ 0777 system system"
        ]
    }]
}
```

## Dependencies from Other Build System Components

### Direct Dependencies on common/

| File | Dependency | Purpose |
|------|------------|---------|
| `config/BUILD.gn:273` | `//build/common/ubsan:ubsan` | UBSan runtime for non-sanitizer builds |
| `templates/idl/ohos_idl.gni:48` | `//build/common/musl:musl-libcxx.so` | C++ runtime for IDL compiler |
| `ohos_var.gni:14` | `import("//build/common.gni")` | Common build variables |
| `lite/ohos_var.gni:13` | `import("//build/common.gni")` | Common build variables (Lite) |
| `config/compiler/BUILD.gn:13` | `import("//build/common.gni")` | Common compiler settings |

### Dependencies on common_packages

The `common_packages` group is the primary interface. It is typically depended on by:
- Image generation targets
- Full system build targets
- SDK packaging targets

### Template Dependencies

Various templates in `//build/templates/` depend on scripts and utilities from `//build/templates/common/` (note: this is a different directory from `//build/common/`):
- `get_subsystem_name.py` - Used by cxx.gni, rust templates, app.gni
- `check_target.gni` - Used by cxx.gni, rust templates
- `copy.gni` - Used by bpf, ace, prebuilt templates
- `collect_target.gni` - Used by abc, bpf, rust, cxx templates

## Key Concepts

### Image Installation

All common components specify `install_images` to control which images they are included in:
- **system**: Main system partition
- **vendor**: Vendor partition
- **ramdisk**: Initial boot ramdisk
- **updater**: OTA update ramdisk
- **system_arm64e**: ARM64E variant system partition

### Sanitizer Selection Logic

The build system selects sanitizer runtime based on build flags:

1. **ASan build** (`is_asan = true`):
   - Includes ASan runtime libraries
   - Includes asan.cfg init config
   - Uses ASan-instrumented musl linker

2. **TSan build** (`is_tsan = true`):
   - Includes TSan runtime libraries
   - Includes asan.cfg (shared config)

3. **Regular build** (no sanitizers):
   - Includes UBSan runtime libraries
   - Includes ubsan.cfg init config

### Architecture Support

| Component | arm | arm64 | x86_64 | riscv64 |
|-----------|-----|-------|--------|---------|
| libc++_shared.so | ✓ | ✓ | ✓ | - |
| musl linker | ✓ | ✓ | - | ✓ |
| ASan runtime | ✓ | ✓ | ✓ | ✓ |
| HWASan runtime | - | ✓ | - | - |
| TSan runtime | - | ✓ | - | - |
| UBSan runtime | ✓ | ✓ | ✓ | ✓ |

### Inner API Tags

Components declare their API stability level:
- `platformsdk`: Platform SDK API
- `chipsetsdk`: Chipset SDK API
- `chipsetsdk_sp`: Chipset SDK Service Provider API

## Build Integration

### Typical Usage Flow

1. **Build initialization**: `common.gni` imported for common variables
2. **Target definition**: Individual modules may depend on specific common targets
3. **Image generation**: `common_packages` group is included in image dependencies
4. **Installation**: Prebuilt binaries are copied to appropriate image directories

### Sanitizer Build Flow

1. Developer sets `is_asan=true` or `is_tsan=true` in GN args
2. `common/BUILD.gn` selects appropriate sanitizer libraries
3. Sanitizer init configs (asan.cfg) are installed to `etc/init/`
4. At boot, init system runs commands from cfg to set up environment
5. Runtime options loaded from `/system/etc/asan.options`

## Summary

The `common/` directory is a critical component of the OpenHarmony build system that provides:

1. **Runtime Libraries**: C++ standard library (libc++), musl libc linker
2. **Sanitizer Support**: ASan, HWASan, TSan, UBSan runtimes with configurations
3. **System Configuration**: Library search paths, sanitizer init scripts
4. **Multi-Architecture Support**: arm, arm64, x86_64, riscv64
5. **Image Integration**: Installation to system, vendor, ramdisk, updater images

These components are foundational - without them, no OpenHarmony system could boot or run C/C++ applications. They bridge the gap between the toolchain (Clang) and the runtime environment (musl libc + OpenHarmony init system).
