# core/ - OpenHarmony Build System Core

**Location:** `/home/zhuojw/a_git/ohos-mani-v2/build/core`

## Overall Responsibility

The `core/` directory is the **central entry point and configuration hub** for the OpenHarmony GN-based build system. It serves as the bridge between the `hb` build tool and the underlying GN/Ninja build engine, defining:

1. **Build Configuration Root** - The `.gn` file that points to the master build configuration
2. **Build Entry Points** - GN targets that orchestrate the entire build process
3. **Security & Sandboxing** - Script execution allowlist for build safety
4. **Build Utilities** - Supporting scripts for build verification

The core/ directory is loaded when GN starts and configures the entire build context for all other build files in the system.

---

## Directory Structure

```
core/
├── gn/
│   ├── BUILD.gn                    # Main build targets and orchestration
│   ├── dotfile.gn                  # GN dotfile - build system entry point
│   └── ohos_exec_script_allowlist.gni  # Security: allowed script execution paths
└── build_scripts/
    └── verify_notice.sh            # License notice verification script
```

---

## 1. GN Dotfile Configuration

### File: `gn/dotfile.gn`

This is the **GN dotfile** (equivalent to `.gn` at repository root) that GN reads first when starting the build.

**Key Configurations:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `buildconfig` | `//build/config/BUILDCONFIG.gn` | Master build configuration file |
| `root` | `//build/core/gn` | Source root location for build system |
| `script_executable` | `/usr/bin/env` | Default interpreter for action/exec_script |
| `ohos_components_support` | `true` | Enable OpenHarmony component system |
| `exec_script_whitelist` | (from `ohos_exec_script_config`) | Security allowlist for script execution |

**Import Chain:**
```
dotfile.gn
    ↓ (imports)
ohos_exec_script_allowlist.gni
    ↓ (defines)
ohos_exec_script_config.exec_script_allowlist
```

---

## 2. Build Entry Points (BUILD.gn)

### File: `gn/BUILD.gn`

This is the **root BUILD.gn** that defines the top-level build targets. It's the main entry point for the build process after GN initialization.

### Build Targets Overview

#### Target Selection Logic

The build selects targets based on `product_name`:

```
if (product_name == "ohos-sdk"):
    → build_ohos_sdk
else if (product_name == "arkui-x"):
    → arkui_targets
else:
    → make_all (standard system build)
```

### Primary Build Targets

#### 1. `build_ohos_sdk` (for SDK builds)

**Purpose:** Build the OpenHarmony SDK package

**Dependencies:**
- `//build/ohos/ndk:ohos_ndk` - Native Development Kit
- `//build/ohos/sdk:ohos_sdk` - SDK generation
- `//build/ohos/sdk:ohos_sdk_verify` - SDK verification

#### 2. `arkui_targets` (for ArkUI-X builds)

**Purpose:** Build cross-platform ArkUI-X SDK

**Dependencies:**
- `//build_plugins/sdk:arkui_cross_sdk`

#### 3. `make_all` (standard system build)

**Purpose:** Main target for building OpenHarmony system images

**Dependencies:**
- `:make_inner_kits` - Build internal component interfaces
- `:packages` - Package generation
- `:images` (conditional) - System image creation (standard system only)

**Condition:**
```gn
if (is_standard_system && !is_llvm_build) {
  deps += [ ":images" ]
}
```

### Sub-Targets

| Target | Purpose | Condition |
|--------|---------|-----------|
| `:images` | Create system images | `!is_llvm_build` |
| `:packages` | Generate installable packages | Always |
| `:make_inner_kits` | Build component inner interfaces | Always |
| `:build_all_test_pkg` | Build test packages | `testonly = true` |
| `:make_test` | Test case packaging | `testonly = true` |

### Target Paths

```
:images           → //build/ohos/images:make_images
:packages         → //build/ohos/packages:make_packages
:make_inner_kits  → $root_build_dir/build_configs:inner_kits
:make_test        → //build/ohos/packages:build_all_test_pkg + test packaging
```

---

## 3. Security Configuration

### File: `gn/ohos_exec_script_allowlist.gni`

This file defines a **security allowlist** for script execution during the build process.

**Purpose:** 
- Restrict which BUILD.gn files can use `exec_script()` 
- Prevent arbitrary script execution during builds
- Enhance build reproducibility and security

**Structure:**
```gni
ohos_exec_script_config = {
  exec_script_allowlist = [
    "//arkcompiler/ets_frontend/ts2panda/BUILD.gn",
    "//base/hiviewdfx/hiview/BUILD.gn",
    "//build/config/BUILDCONFIG.gn",
    "//foundation/arkui/ace_engine/ace_config.gni",
    // ... 200+ entries
  ]
}
```

**Categories of Allowed Scripts:**

| Category | Examples |
|----------|----------|
| Build System | `//build/config/BUILDCONFIG.gn`, `//build/ohos_var.gni` |
| Compiler/Toolchain | `//build/toolchain/BUILD.gn`, `//build/config/compiler/*` |
| Templates | `//build/templates/cxx/cxx.gni`, `//build/templates/rust/*` |
| ArkCompiler | `//arkcompiler/ets_frontend/*`, `//arkcompiler/runtime_core/*` |
| Foundation | `//foundation/arkui/*`, `//foundation/communication/*` |
| Third Party | `//third_party/flutter/*`, `//third_party/skia/*` |
| Device/Board | `//device/board/*`, `//device/soc/*` |

**Integration with dotfile.gn:**
```gni
import("//build/core/gn/ohos_exec_script_allowlist.gni")
exec_script_whitelist = ohos_exec_script_config.exec_script_allowlist
```

---

## 4. Build Scripts

### File: `build_scripts/verify_notice.sh`

**Purpose:** Verify NOTICE file format for license compliance

**Usage:**
```bash
verify_notice.sh <notice_file> <output_file> <platform_dir>
```

**Parameters:**
- `$1`: Path to NOTICE file to verify
- `$2`: Output file for verification results ("Success" or "Failed")
- `$3`: Platform directory containing helper files

**Verification Logic:**
1. Check if NOTICE file exists (if not, return Success)
2. Count specific delimiter lines:
   - `====` separator lines
   - "Notices for file(s):" markers
   - `----` divider lines
3. Verify counts match (equal lines must equal file flag lines)
4. Output "Success" or "Failed" to result file

**Used by:** Package generation to validate third-party license notices

---

## 5. Build Process Flow

### Initialization Flow

```
1. hb build command invoked
   ↓
2. hb/main.py → _init_build_module()
   ↓
3. OHOSPreloader.run() - Load product/device configuration
   ↓
4. OHOSLoader.run() - Generate build configs
   ↓
5. Gn.run() - Execute 'gn gen'
   ↓
6. GN reads //build/core/gn/dotfile.gn
   ↓
7. dotfile.gn imports BUILDCONFIG.gn and sets up build context
   ↓
8. GN parses //build/core/gn/BUILD.gn (root BUILD.gn)
   ↓
9. BUILD.gn imports //build/ohos_var.gni
   ↓
10. Targets resolved based on product_name
   ↓
11. Ninja.run() - Execute 'ninja' to build targets
```

### Target Resolution Flow

```
hb build
  ↓
Gn.execute_gn_gen_cmd()
  ↓
gn gen --args="..." <out_path>
  ↓
GN reads dotfile.gn
  ↓
GN loads BUILDCONFIG.gn (buildconfig)
  ↓
GN parses //build/core/gn/BUILD.gn
  ↓
Product-based target selection:
    - ohos-sdk → :build_ohos_sdk
    - arkui-x  → :arkui_targets  
    - default  → :make_all
  ↓
:make_all dependencies:
    ├─ :make_inner_kits
    ├─ :packages
    └─ :images (if standard system)
```

---

## 6. Integration with hb Tool

### hb → core/ Integration Points

| hb Component | core/ File | Purpose |
|--------------|------------|---------|
| `hb build` | `gn/BUILD.gn` | Executes the main build target |
| `hb build` | `gn/dotfile.gn` | GN reads this for build configuration |
| GN service | `gn/dotfile.gn` | `Gn._execute_gn_gen_cmd()` invokes gn |
| Ninja service | All build targets | `Ninja._execute_ninja_cmd()` executes build |

### GN Command Execution

From `hb/services/gn.py`:
```python
def _execute_gn_gen_cmd(self):
    gn_gen_cmd = [
        self.exec, 'gen', 
        '--json=gn_log.json',
        '--args={}'.format(' '.join(self._convert_args())),
        self.config.out_path
    ] + self._convert_flags()
```

This command:
1. Runs `gn gen` with the dotfile at `//build/core/gn`
2. Passes build arguments from hb configuration
3. Generates ninja build files in `out_path`

### Build Target Execution

From `hb/services/ninja.py`:
```python
def _execute_ninja_cmd(self):
    ninja_cmd = [
        self.exec, 
        '-w', 'dupbuild=warn',
        '-C', self.config.out_path
    ] + self._convert_args()
```

Ninja executes the targets defined in `//build/core/gn/BUILD.gn`.

---

## 7. Dependencies

### core/ → Other Build Modules

```
core/gn/BUILD.gn:
  ├─ imports //build/ohos_var.gni (build variables)
  ├─ deps //build/ohos/ndk:ohos_ndk
  ├─ deps //build/ohos/sdk:ohos_sdk
  ├─ deps //build/ohos/images:make_images
  ├─ deps //build/ohos/packages:make_packages
  └─ deps $root_build_dir/build_configs:inner_kits

core/gn/dotfile.gn:
  ├─ imports //build/core/gn/ohos_exec_script_allowlist.gni
  ├─ references //build/config/BUILDCONFIG.gn
  └─ sets root = "//build/core/gn"
```

### Build System Import Chain

```
dotfile.gn
    ↓
BUILDCONFIG.gn (master config)
    ↓
    ├─ //build/common.gni
    ├─ //build/version.gni
    └─ //build/ohos_var.gni (imported by BUILD.gn)
        ↓
        ├─ Product config (from preloader)
        ├─ Device config
        └─ Toolchain config
```

---

## 8. Key Variables and Configurations

### From BUILDCONFIG.gn (referenced by core/)

| Variable | Description | Used In |
|----------|-------------|---------|
| `product_name` | Target product name | BUILD.gn target selection |
| `device_name` | Target device name | Device-specific config |
| `is_standard_system` | Standard OS build flag | Conditional targets |
| `is_llvm_build` | LLVM toolchain build | Skip images if true |
| `is_mini_system` | Mini OS build flag | Lite system config |
| `is_small_system` | Small OS build flag | Lite system config |
| `ohos_indep_compiler_enable` | Independent component build | Feature flags |

### From ohos_var.gni (imported by BUILD.gn)

| Variable | Description | Example |
|----------|-------------|---------|
| `system_base_dir` | System package output dir | "system" |
| `ramdisk_base_dir` | Ramdisk output dir | "ramdisk" |
| `vendor_base_dir` | Vendor package dir | "vendor" |
| `build_ohos_sdk` | Build SDK flag | true/false |
| `build_ohos_ndk` | Build NDK flag | true/false |

---

## 9. File Permissions and Security

### Script Execution Security

The `ohos_exec_script_allowlist.gni` implements a **whitelist approach**:

1. Only listed BUILD.gn files can use `exec_script()`
2. Attempts to execute scripts from non-listed files will fail
3. This prevents malicious or unintended script execution

### Build Sandbox Support

From BUILDCONFIG.gn:
```gni
declare_args() {
  use_sandbox = false
  sandbox_debug = false
}
```

When enabled, action targets are wrapped in a sandbox environment for additional security.

---

## Summary

The `core/` directory is the **foundation of the OpenHarmony build system**:

1. **Entry Point**: `dotfile.gn` is the first file GN reads
2. **Configuration**: Points to master build config (BUILDCONFIG.gn)
3. **Security**: Implements script execution allowlist
4. **Orchestration**: Defines top-level build targets in BUILD.gn
5. **Integration**: Bridges hb tool with GN/Ninja build engine

When you run `hb build`, the execution flows through core/ to initialize the build context, select appropriate targets based on product configuration, and delegate to the underlying build system.
