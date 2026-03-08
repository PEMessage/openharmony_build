# OpenHarmony Build System - ohos/ Directory Codemap

## Overview

The `ohos/` directory is the core of OpenHarmony-specific build configurations, templates, and processing flows. It contains the GN (Generate Ninja) build templates, Python scripts, and configuration files that define how OpenHarmony components are built, packaged, and assembled.

## Directory Structure

```
ohos/
├── ace/              # ArkUI/ACE framework build support
├── app/              # HAP (HarmonyOS Ability Package) build templates
├── common/           # Common build utilities and subsystem merging
├── ebpf.gni          # eBPF (Extended Berkeley Packet Filter) build template
├── hisysevent/       # HiSysEvent configuration processing
├── images/           # System image generation
├── kernel/           # Kernel version configuration
├── kits/             # InnerKits (internal API) checking
├── native_stub/      # Native stub library generation
├── ndk/              # NDK (Native Development Kit) build support
├── notice/           # License/notice file collection
├── ohos_kits.gni     # Main kits template definition
├── ohos_part.gni     # OpenHarmony part template
├── ohos_test.gni     # Test framework template
├── packages/         # Package assembly and installation
├── sa_profile/       # SA (System Ability) profile processing
├── sbom/             # SBOM (Software Bill of Materials) generation
├── sdk/              # SDK packaging
├── statistics/       # Build statistics
├── taihe_idl/        # Taihe IDL (Interface Definition Language) support
├── testfwk/          # Test framework utilities
└── update/           # OTA update package generation
```

## Key Components

### 1. Core Templates (.gni files)

#### `ohos_part.gni`
- **Purpose**: Defines the `ohos_part` template - the fundamental building block for OpenHarmony parts
- **Key Features**:
  - Aggregates module lists into parts
  - Handles SDK dependencies
  - Generates part installation info
  - Supports variant handling (phone, etc.)
  - Integrates with eBPF testing

#### `ohos_kits.gni`
- **Purpose**: Defines `ohos_inner_kits` for SDK library management
- **Key Features**:
  - Manages SDK libraries (SO, JAR files)
  - Handles header file configurations
  - Supports prebuilt libraries
  - Interface compatibility checking

#### `ohos_test.gni`
- **Purpose**: Test aggregation template
- **Key Features**:
  - Groups test packages by part
  - Manages test dependencies

#### `ebpf.gni`
- **Purpose**: eBPF test case collection
- **Key Features**:
  - Collects eBPF test cases
  - Generates test configurations

### 2. Subsystem Directories

#### `kits/`
- **`kits_check.gni`**: Template for checking InnerKits interface compatibility
- **`kits_check_remove.py`**: Validates that SDK libraries haven't been removed
- **Purpose**: Ensures API stability by checking against saved signatures

#### `ndk/`
- **`ndk.gni`**: Main NDK build template with 615 lines
  - `ohos_ndk_library`: Generate NDK stub libraries
  - `ohos_ndk_headers`: Handle NDK header files
  - `ohos_ndk_copy`: Copy NDK files
  - `ohos_ndk_toolchains`: NDK toolchain management
  - `ohos_ndk_prebuilt_library`: Prebuilt NDK libraries
  - `current_ndk`: Current NDK target
- **Python Scripts**:
  - `generate_ndk_stub_file.py`: Generate stub C files from JSON description
  - `generate_version_script.py`: Generate linker version scripts
  - `check_ndk_header.py`: Validate NDK headers compile
  - `check_ndk_header_signature.py`: Check header signature compatibility
  - `collect_ndk_syscap.py`: Collect system capability info
  - `archive_ndk.py`: Archive NDK into zip files
  - `generate_ndk_docs.py`: Generate Doxygen documentation
  - `create_ndk_docs_portal.py`: Create documentation portal
  - `copy_notices_file.py`: Copy NDK notice files
  - `parse_ndk_targets.py`: Parse NDK targets from BUILD.gn
  - `scan_ndk_targets.py`: Scan all NDK targets in repository
- **`BUILD.gn`**: Main NDK build file (439 lines)
- **`cmake/`**: CMake toolchain files for NDK

#### `sdk/`
- **`sdk.gni`**: SDK packaging template (405 lines)
  - `copy_and_archive`: Copy and archive SDK modules
  - `make_sdk_modules`: Create SDK module packages
  - `make_linux_sdk_modules`: Linux-specific SDK
  - `make_windows_sdk_modules`: Windows-specific SDK
  - `make_darwin_sdk_modules`: macOS-specific SDK
  - `make_ohos_sdk_modules`: OpenHarmony-specific SDK
  - `current_sdk`: Current SDK target
- **Python Scripts**:
  - `parse_sdk_description.py`: Parse SDK description JSON
  - `generate_all_types_sdk.py`: Generate SDK build files
  - `copy_sdk_modules.py`: Copy SDK modules
  - `check_sdk_completeness.py`: Verify SDK completeness
  - `add_notice_file.py`: Add notice files to SDK archives
  - `convert_permissions.py`: Convert permission definitions
  - `parse_interface_sdk.py`: Parse interface SDK
  - `parse_public_sdk.py`: Parse public SDK
  - `generate_hap_build_sdk_config.py`: Generate HAP build SDK config
  - `remove_cangjie_ohos_sdk_config.py`: Remove Cangjie SDK config
  - `parse_description.py`: Parse SDK description
- **Configuration Files**:
  - `ohos_sdk_description_std.json`: Standard SDK description
  - `sdk_delivery_list.json`: SDK delivery check list
  - `type_to_display_name.json`: SDK type display names
  - `variant_to_product.json`: Variant to product mapping

#### `app/`
- **`app.gni`**: HAP build template (855 lines)
  - `ohos_app_scope`: App scope configuration
  - `ohos_assets`: Asset management
  - `ohos_js_assets`: JavaScript assets
  - `ohos_resources`: Resource compilation
  - `ohos_app`: Main app target
  - `ohos_hap`: HAP package generation
- **`app_internal.gni`**: Internal app templates (751 lines)
  - `merge_profile`: Merge app profiles
  - `compile_resources`: Resource compilation
  - `package_app`: App packaging
  - `app_sign`: App signing

#### `sa_profile/` (System Ability Profiles)
- **`sa_profile.gni`**: SA profile template (242 lines)
  - `ohos_sa_profile`: Define SA profiles
  - `ohos_sa_install_info`: SA installation info
  - `ohos_sa_info_archive`: SA archive processing
- **Python Scripts**:
  - `sa_profile.py`: Generate SA info files
  - `sa_profile_binary.py`: Process binary SA profiles
  - `sa_profile_merge.py`: Merge SA profiles
  - `sa_profile_source.py`: Process source SA profiles
  - `sa_profile_archive.py`: Archive SA profiles
  - `src_sa_profile_process.py`: Process SA profiles by variant
- **`sa_info_process/`**:
  - `merge_sa_info.py`: Merge SA info JSON files
  - `sort_sa_by_bootphase.py`: Sort SAs by boot phase
  - `sa_info_config_errors.py`: Error definitions

#### `notice/`
- **`notice.gni`**: Notice collection template (138 lines)
  - `collect_notice`: Collect module notice files
- **Python Scripts**:
  - `collect_module_notice_file.py`: Collect individual module notices
  - `collect_system_notice_files.py`: Collect system-wide notices
  - `merge_notice_files.py`: Merge notice files into final output

#### `images/`
- **`BUILD.gn`**: Image build configuration (430 lines)
  - Defines system, vendor, userdata, ramdisk, updater images
- **Python Scripts**:
  - `build_image.py`: Main image building script (191 lines)
  - `get_module_install_dest.py`: Calculate module install destinations
  - `adlt_wrapper.py`: ADLT (Allowed Dynamic Link Targets) wrapper
- **`mkimage/`**:
  - `mkimages.py`: Main image creation script (146 lines)
  - `mkextimage.py`: ext4 image creation (140 lines)
  - `mkf2fsimage.py`: F2FS image creation (155 lines)
  - `mkcpioimage.py`: CPIO ramdisk creation (140 lines)
  - `mkchip_ckm.py`: Chip CKM image creation
  - `imkcovert.py`: Image format conversion (sparse/unsparse)
  - `judge_updater_image.py`: Validate updater image dependencies
  - Configuration files: `*_image_conf.txt` for each image type

#### `packages/`
- **`BUILD.gn`**: Package assembly configuration (430+ lines)
- **Python Scripts**:
  - `modules_install.py`: Install modules to system (347 lines)
  - `parts_install_info.py`: Generate parts installation info
  - `resources_collect.py`: Collect test resources
  - `system_notice_info.py`: Collect system notice info
  - `fs_process.py`: Filesystem processing
  - `gen_required_modules_list.py`: Generate required modules list
  - `generate_host_symlink.py`: Generate host symlinks
  - `system_gzip_package.py`: Create gzip packages
  - `system_z_package.py`: Create Z-compressed packages
  - `process_field_validate.py`: Validate process fields
  - `check_seccomp_library_name.py`: Check seccomp library names
  - `backup_restore_artifact.py`: Backup/restore artifacts
  - `bootpath_collection.py`: Collect boot paths
  - `kernel_permission.py`: Kernel permission handling
  - `platforms_install_info.py`: Platform installation info
- **`rules/`**:
  - `categorized_libraries_utils.py`: Library categorization
  - `categorized-libraries.json`: Library categories

#### `testfwk/`
- **`gen_module_list_files.py`: Generate test module list files
- **`testcase_resource_copy.py`: Copy test case resources (349 lines)
- **`test_js_file_copy.py`: Copy JavaScript test files
- **`test_js_stage_file_copy.py`: Copy JavaScript stage test files
- **`test_py_file_copy.py`: Copy Python test files
- **`fuzz_config_file_copy.py`: Copy fuzz test config files
- **`arkts_tdd_cases_build.py`: Build ArkTS TDD test cases

#### `hisysevent/`
- **`hisysevent.gni`**: HiSysEvent template (64 lines)
  - `ohos_hisysevent_install_info`: Process HiSysEvent configs
- **`hisysevent_process.py`: Process HiSysEvent configurations
- **`gen_def_from_all_yaml.py`: Generate definitions from YAML (780 lines)

#### `common/`
- **`BUILD.gn`**: Common build targets (126 lines)
  - Generates source/binary install info
  - Merges subsystem information
- **`merge_all_subsystem.py`: Merge all subsystem info
- **`binary_install_info.py`: Binary installation info

#### `kernel/`
- **`kernel.gni`**: Kernel version configuration
  - Defines `linux_kernel_version` (default: "linux-6.6")

#### `native_stub/`
- **`native_stub.gni`**: Native stub library template (339 lines)
  - `ohos_native_stub_library`: Generate stub libraries
  - `ohos_native_stub_versionscript`: Generate version scripts
  - `ohos_native_stub_headers`: Process stub headers

#### `taihe_idl/`
- **`taihe.gni`**: Taihe IDL template (103 lines)
  - `ohos_taihe`: Compile Taihe IDL files
  - `copy_taihe_idl`: Copy Taihe IDL files
  - `taihe_shared_library`: Build Taihe shared libraries
- **`taihe_retry.sh`**: Retry script for Taihe compilation

#### `update/`
- **`check_abi_and_copy_deps.py`: Check ABI compatibility and copy dependencies (240 lines)

#### `statistics/`
- **`build_overlap_statistics.py`: Calculate build overlap statistics (160 lines)

#### `ace/`
- **`ace.gni`**: ACE framework template (68 lines)
  - `js_declaration`: JS declaration handling
  - `gen_js_obj`: Generate JS object files
- **`ace_args.gni`**: ACE arguments configuration

#### `sbom/` (Software Bill of Materials)
Comprehensive SBOM generation system:
- **`generate_sbom.py`**: Main SBOM generation entry (154 lines)
- **`README_zh.md`**: Detailed Chinese documentation (463 lines)

**Directory Structure:**
- **`analysis/`**: Dependency analysis
  - `depend_graph.py`: Dependency graph analyzer (254 lines)
  - `file_dependency.py`: File dependency analyzer (589 lines)
  - `install_module.py`: Install module analyzer (138 lines)
  - `project_dependency.py`: Project dependency analyzer (171 lines)

- **`common/`**: Common utilities
  - `utils.py`: Utility functions (196 lines)

- **`converters/`**: SBOM format converters
  - `api.py`: Converter API (65 lines)
  - `base.py`: Base converter classes (104 lines)
  - `spdx23.py`: SPDX 2.3 converter (252 lines)

- **`data/`**: Data models
  - `build_setting.py`: Build settings (33 lines)
  - `file_dependence.py`: File dependencies (327 lines)
  - `manifest.py`: Manifest parser (249 lines)
  - `ninja_json.py`: Ninja JSON model (59 lines)
  - `opensource.py`: OpenSource metadata (81 lines)
  - `project_dependence.py`: Project dependencies (69 lines)
  - `target.py`: Build target model (55 lines)

- **`extraction/`**: Resource extraction
  - `local_resource_loader.py`: Resource loader (318 lines)
  - `copyright_and_license_scanner.py`: License scanner (349 lines)

- **`pipeline/`**: SBOM generation pipeline
  - `sbom_generator.py`: Main SBOM generator (377 lines)

- **`sbom/`**: SBOM builders
  - **`builder/`**: Builder classes
    - `base_builder.py`: Base builder (203 lines)
    - `document_builder.py`: Document builder (341 lines)
    - `file_builder.py`: File builder (225 lines)
    - `package_builder.py`: Package builder (281 lines)
    - `relationship_builder.py`: Relationship builder (130 lines)
    - `sbom_meta_data_builder.py`: Metadata builder (283 lines)
  - **`config/`**: Configuration
    - `field_config.py`: Field configuration manager (169 lines)
    - **`configs/`**: JSON configs
      - `document.config.json`: Document fields
      - `file.config.json`: File fields
      - `package.config.json`: Package fields
      - `relationship.config.json`: Relationship fields
  - **`metadata/`**: Metadata models
    - `sbom_meta_data.py`: Core SBOM metadata (430 lines)
  - **`validation/`**: Validation
    - `validator.py`: Field validator (71 lines)

## Build Process Flow

### 1. Part Definition Phase
```
ohos_part("part_name") {
  subsystem_name = "..."
  module_list = ["..."]
}
```
- Parts aggregate modules
- Generate part info JSON
- Handle SDK dependencies

### 2. Module Building Phase
- Build individual targets
- Generate module_info.json for each target
- Collect metadata for installation

### 3. Installation Phase
```
packages/modules_install.py
```
- Reads all module_info.json files
- Determines install paths
- Creates install manifests

### 4. Image Generation Phase
```
images/build_image.py
```
- Collects installed files
- Creates filesystem images
- Supports ext4, f2fs, cpio formats

### 5. SDK Packaging Phase
```
sdk/copy_sdk_modules.py
sdk/parse_sdk_description.py
```
- Parses SDK description
- Copies modules to SDK structure
- Creates platform-specific archives

### 6. SBOM Generation Phase (Optional)
```
sbom/generate_sbom.py
```
- Analyzes build dependencies
- Generates SPDX 2.3 format
- Tracks file and package relationships

## Integration with Broader Build System

### Input Dependencies
- `//build/ohos_var.gni`: Global OpenHarmony variables
- `//build/config/python.gni`: Python action templates
- `${root_build_dir}/build_configs/`: Generated configuration files

### Output Artifacts
- `${root_build_dir}/packages/`: Package output
- `${root_build_dir}/NOTICE_FILES/`: License files
- `${root_build_dir}/sbom/`: SBOM files (when enabled)
- `out/{product}/images/`: System images

### Key Integration Points
1. **GN Templates**: Define reusable build patterns
2. **Python Scripts**: Handle complex build logic
3. **Metadata Files**: JSON-based information exchange
4. **Configuration**: GNI files for build variants

## Usage Patterns

### Defining a Part
```gn
import("//build/ohos/ohos_part.gni")

ohos_part("my_part") {
  subsystem_name = "my_subsystem"
  module_list = [
    "//path/to/module1:target1",
    "//path/to/module2:target2",
  ]
}
```

### Defining InnerKits
```gn
import("//build/ohos/ohos_kits.gni")

ohos_inner_kits("my_sdk") {
  sdk_libs = [
    {
      type = "so"
      name = "//path/to/lib:libname"
      header = {
        header_files = ["header.h"]
        header_base = "path/to/include"
      }
    },
  ]
}
```

### Building an App
```gn
import("//build/ohos/app/app.gni")

ohos_hap("my_app") {
  hap_profile = "./module.json"
  sources = ["..."]
}
```

### Generating SBOM
```bash
./build.sh --product-name {product} --sbom=true
```

## Key Design Principles

1. **Modularity**: Each subdirectory handles a specific concern
2. **Composability**: Templates can be combined and extended
3. **Configuration-Driven**: JSON/GNI files control behavior
4. **Metadata-Rich**: Build artifacts carry installation metadata
5. **Multi-Platform**: Support for Linux, Windows, macOS SDK builds
6. **Compliance**: SBOM generation for security and license compliance
