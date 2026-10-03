cmake_minimum_required(VERSION 4.4 FATAL_ERROR)

set(CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_LIST_DIR}/vcpkg/scripts/buildsystems/vcpkg.cmake")

if(NOT DEFINED CACHE{VCPKG_BOOTSTRAP_OPTIONS})
    list(APPEND VCPKG_BOOTSTRAP_OPTIONS "-disableMetrics")
endif()

if(NOT DEFINED CACHE{VCPKG_INSTALL_OPTIONS})
    list(APPEND VCPKG_INSTALL_OPTIONS "--x-buildtrees-root=${CMAKE_CURRENT_BINARY_DIR}/vcpkg/buildtrees")
    list(APPEND VCPKG_INSTALL_OPTIONS "--x-downloads-root=${CMAKE_CURRENT_BINARY_DIR}/vcpkg/downloads")
    list(APPEND VCPKG_INSTALL_OPTIONS "--x-packages-root=${CMAKE_CURRENT_BINARY_DIR}/vcpkg/packages")
endif()

if(NOT DEFINED CACHE{VCPKG_INSTALLED_DIR})
    set(VCPKG_INSTALLED_DIR "${CMAKE_CURRENT_BINARY_DIR}/vcpkg/installed")
endif()

project(
    WASMEDGE
    VERSION 0.18.0
    COMPAT_VERSION 0.18
    SPDX_LICENSE Apache-2.0
    DESCRIPTION "WasmEdge is a lightweight, high-performance, and extensible WebAssembly runtime"
    HOMEPAGE_URL "https://wasmedge.org/"
    LANGUAGES C CXX
)

option(BUILD_SHARED_LIBS "Build using shared libraries" OFF)
include(CTest)
include(FeatureSummary)
include(GNUInstallDirs)

set(WASMEDGE_API_VERSION "5" CACHE STRING "WasmEdge Plugin API version")
set(WASMEDGE_CAPI_VERSION "0.1.1" CACHE STRING "WasmEdge C API library version")
set(WASMEDGE_CAPI_SOVERSION "0" CACHE STRING "WasmEdge C API library soversion")

option(WASMEDGE_BUILD_TOOLS "Generate wasmedge and wasmedgec tools." ON)
option(WASMEDGE_USE_LLVM "Enable WasmEdge LLVM-based compilation runtime." ON)

option(WASMEDGE_BUILD_PLUGINS "Generate plugins." ON)
option(WASMEDGE_BUILD_WASI_NN_RPC "Generate WASI-NN RPC." OFF)

option(WASMEDGE_BUILD_EXAMPLE "Generate examples." OFF)

option(WASMEDGE_BUILD_TESTS "Generate build targets for the wasmedge unit tests." OFF)

option(WASMEDGE_BUILD_FUZZING "Generate fuzzing test tools." OFF)

add_feature_info("WasmEdge JIT/AOT" WASMEDGE_USE_LLVM "Enable WasmEdge LLVM-based compilation runtime.")
add_feature_info("WasmEdge Plugins" WASMEDGE_BUILD_PLUGINS "Generate plugins.")

find_package(spdlog CONFIG)
find_package(LLVM CONFIG)
find_package(blake3 CONFIG)

set_package_properties(spdlog PROPERTIES TYPE REQUIRED)
if(WASMEDGE_USE_LLVM)
    set_package_properties(LLVM PROPERTIES TYPE REQUIRED)
    set_package_properties(blake3 PROPERTIES TYPE REQUIRED)
endif()

feature_summary(
  WHAT ALL
  INCLUDE_QUIET_PACKAGES
  DESCRIPTION "Feature summary:"
  FATAL_ON_MISSING_REQUIRED_PACKAGES
)

if(WASMEDGE_USE_LLVM)
    list(APPEND CMAKE_MODULE_PATH "${LLVM_CMAKE_DIR}")
    include(HandleLLVMOptions)
    add_definitions(${LLVM_DEFINITIONS})
endif()

# ==
# include/
# ==

set(CPACK_PACKAGE_VERSION "${PROJECT_VERSION}")

include(CheckCXXSymbolExists)
check_cxx_symbol_exists(mmap sys/mman.h HAVE_MMAP)
include(CheckIncludeFileCXX)
check_include_file_cxx(pwd.h HAVE_PWD_H)

# ==
# lib/common
# ==
configure_file(include/common/config.h.in include/common/config.h)
configure_file(include/common/version.h.in include/common/version.h)

add_library(WASMEDGE_common)
add_library(WASMEDGE::common ALIAS WASMEDGE_common)
target_sources(WASMEDGE_common
    PRIVATE
        lib/common/hash.cpp
        lib/common/hexstr.cpp
        lib/common/spdlog.cpp
        lib/common/errinfo.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
            "${PROJECT_BINARY_DIR}/include"
        FILES
            include/common/async.h
            include/common/config.h.in
            include/common/configure.h
            include/common/defines.h
            include/common/dense_enum_map.h
            include/common/endian.h
            include/common/enum_ast.hpp
            include/common/enum_configure.h
            include/common/enum_configure.hpp
            include/common/enum_errcode.h
            include/common/enum_errcode.hpp
            include/common/enum_errinfo.hpp
            include/common/enum_types.h
            include/common/enum_types.hpp
            include/common/enum.inc
            include/common/errcode.h
            include/common/errinfo.h
            include/common/executable.h
            include/common/expected.h
            include/common/filesystem.h
            include/common/fmt.h
            include/common/hash.h
            include/common/hexstr.h
            include/common/int128.h
            include/common/roundeven.h
            include/common/span.h
            include/common/spare_enum_map.h
            include/common/spdlog.h
            include/common/statistics.h
            include/common/symbol.h
            include/common/timer.h
            include/common/types.h
            include/common/variant.h
            include/common/version.h.in
            "${PROJECT_BINARY_DIR}/include/common/config.h"
            "${PROJECT_BINARY_DIR}/include/common/version.h"
)
target_link_libraries(WASMEDGE_common
    PUBLIC
        spdlog::spdlog
)
target_compile_features(WASMEDGE_common
    PUBLIC
        cxx_std_17
)

# ==
# lib/system
# ==
add_library(WASMEDGE_system)
add_library(WASMEDGE::system ALIAS WASMEDGE_system)
target_sources(WASMEDGE_system
    PRIVATE
        lib/system/allocator.cpp
        lib/system/fault.cpp
        lib/system/mmap.cpp
        lib/system/path.cpp
        lib/system/stacktrace.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/system/allocator.h
            include/system/fault.h
            include/system/mmap.h
            include/system/path.h
            include/system/stacktrace.h
            include/system/winapi.h
)
target_link_libraries(WASMEDGE_system
    PUBLIC
        WASMEDGE::common
)
target_compile_features(WASMEDGE_system
    PUBLIC
        cxx_std_17
)

# ==
# lib/po
# ==
add_library(WASMEDGE_po)
add_library(WASMEDGE::po ALIAS WASMEDGE_po)
target_sources(WASMEDGE_po
    PRIVATE
        lib/po/argument_parser.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/po/argument_parser.h
            include/po/error.h
            include/po/helper.h
            include/po/list.h
            include/po/option.h
            include/po/parser.h
            include/po/subcommand.h
)
target_link_libraries(WASMEDGE_po
    PUBLIC
        WASMEDGE::common

)
target_compile_features(WASMEDGE_po
    PUBLIC
        cxx_std_17
)

# ==
# lib/loader
# ==
add_library(WASMEDGE_loaderfilemgr)
add_library(WASMEDGE::loaderfilemgr ALIAS WASMEDGE_loaderfilemgr)
target_sources(WASMEDGE_loaderfilemgr
    PRIVATE
        lib/loader/filemgr.cpp
        lib/loader/shared_library.cpp
        lib/loader/aot_section.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/loader/aot_section.h
            include/loader/filemgr.h
            include/loader/shared_library.h
)
target_link_libraries(WASMEDGE_loaderfilemgr
    PUBLIC
        WASMEDGE::common
        WASMEDGE::system

)
target_compile_features(WASMEDGE_loaderfilemgr
    PUBLIC
        cxx_std_17
)

add_library(WASMEDGE_loader)
add_library(WASMEDGE::loader ALIAS WASMEDGE_loader)
target_sources(WASMEDGE_loader
    PRIVATE
        lib/loader/ast/component/component.cpp
        lib/loader/ast/component/component_section.cpp
        lib/loader/ast/component/component_instance.cpp
        lib/loader/ast/component/component_sort.cpp
        lib/loader/ast/component/component_alias.cpp
        lib/loader/ast/component/component_canonical.cpp
        lib/loader/ast/component/component_start.cpp
        lib/loader/ast/component/component_type.cpp
        lib/loader/ast/component/component_import.cpp
        lib/loader/ast/component/component_export.cpp
        lib/loader/ast/component/component_value.cpp
        lib/loader/ast/component/component_declarator.cpp
        lib/loader/ast/component/component_descriptor.cpp
        lib/loader/ast/component/component_valtype.cpp
        lib/loader/ast/module.cpp
        lib/loader/ast/section.cpp
        lib/loader/ast/description.cpp
        lib/loader/ast/segment.cpp
        lib/loader/ast/type.cpp
        lib/loader/ast/expression.cpp
        lib/loader/ast/instruction.cpp
        lib/loader/serialize/serial_description.cpp
        lib/loader/serialize/serial_expression.cpp
        lib/loader/serialize/serial_instruction.cpp
        lib/loader/serialize/serial_module.cpp
        lib/loader/serialize/serial_section.cpp
        lib/loader/serialize/serial_segment.cpp
        lib/loader/serialize/serial_type.cpp
        lib/loader/loader.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/loader/loader.h
            include/loader/serialize.h
)
target_link_libraries(WASMEDGE_loader
    PUBLIC
        WASMEDGE::common
        WASMEDGE::loaderfilemgr

)
target_compile_features(WASMEDGE_loader
    PUBLIC
        cxx_std_17
)

# ==
# lib/plugin
# ==
add_library(WASMEDGE_plugin)
add_library(WASMEDGE::plugin ALIAS WASMEDGE_plugin)
target_sources(WASMEDGE_plugin
    PRIVATE
        lib/plugin/plugin.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/plugin/plugin.h
    PRIVATE
        FILE_SET private_headers
        TYPE HEADERS
        BASE_DIRS
            include/api/
            "${PROJECT_BINARY_DIR}/include/api"
        FILES
            include/api/wasmedge/wasmedge.h
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/enum_configure.h"
)
target_link_libraries(WASMEDGE_plugin
    PUBLIC
        # WASMEDGE::api
        WASMEDGE::common
        WASMEDGE::loaderfilemgr
        WASMEDGE::po

)
target_compile_features(WASMEDGE_plugin
    PUBLIC
        cxx_std_17
)

# ==
# lib/validator
# ==
add_library(WASMEDGE_validator)
add_library(WASMEDGE::validator ALIAS WASMEDGE_validator)
target_sources(WASMEDGE_validator
    PRIVATE
        lib/validator/component_canon.cpp
        lib/validator/component_context.cpp
        lib/validator/component_name.cpp
        lib/validator/component_type.cpp
        lib/validator/component_types.cpp
        lib/validator/component_validator.cpp
        lib/validator/formchecker.cpp
        lib/validator/validator.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/validator/component_context.h
            include/validator/component_name.h
            include/validator/component_types.h
            include/validator/component_value_decode.h
            include/validator/formchecker.h
            include/validator/validator.h
)
target_link_libraries(WASMEDGE_validator
    PUBLIC
        WASMEDGE::common
)
target_compile_features(WASMEDGE_validator
    PUBLIC
        cxx_std_17
)

# ==
# lib/executor
# ==
add_library(WASMEDGE_executor)
add_library(WASMEDGE::executor ALIAS WASMEDGE_executor)
target_sources(WASMEDGE_executor
    PRIVATE
        lib/executor/instantiate/import.cpp
        lib/executor/instantiate/function.cpp
        lib/executor/instantiate/global.cpp
        lib/executor/instantiate/table.cpp
        lib/executor/instantiate/memory.cpp
        lib/executor/instantiate/elem.cpp
        lib/executor/instantiate/data.cpp
        lib/executor/instantiate/export.cpp
        lib/executor/instantiate/tag.cpp
        lib/executor/instantiate/module.cpp
        lib/executor/instantiate/component/component.cpp
        lib/executor/instantiate/component/component_alias.cpp
        lib/executor/instantiate/component/component_canon.cpp
        lib/executor/instantiate/component/component_canon_abi.cpp
        lib/executor/instantiate/component/component_lower_thunk.cpp
        lib/executor/instantiate/component/component_export.cpp
        lib/executor/instantiate/component/component_import.cpp
        lib/executor/instantiate/component/component_instance.cpp
        lib/executor/instantiate/component/component_module.cpp
        lib/executor/instantiate/component/component_start.cpp
        lib/executor/instantiate/component/component_type.cpp
        lib/executor/engine/proxy.cpp
        lib/executor/engine/controlInstr.cpp
        lib/executor/engine/tableInstr.cpp
        lib/executor/engine/threadInstr.cpp
        lib/executor/engine/memoryInstr.cpp
        lib/executor/engine/variableInstr.cpp
        lib/executor/engine/refInstr.cpp
        lib/executor/engine/engine.cpp
        lib/executor/helper.cpp
        lib/executor/executor.cpp
        lib/executor/coredump.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
        include/executor/component/canonical_abi.h
        include/executor/component/lower_thunk.h
        include/executor/engine/atomic.ipp
        include/executor/engine/binary_numeric_vector_msvc.ipp
        include/executor/engine/binary_numeric_vector.ipp
        include/executor/engine/binary_numeric.ipp
        include/executor/engine/cast_numeric.ipp
        include/executor/engine/memory.ipp
        include/executor/engine/relation_numeric.ipp
        include/executor/engine/simd_ops.h
        include/executor/engine/unary_numeric_vector_msvc.ipp
        include/executor/engine/unary_numeric_vector.ipp
        include/executor/engine/unary_numeric.ipp
        include/executor/engine/vector_helper.h
        include/executor/coredump.h
        include/executor/executor.h
)
target_link_libraries(WASMEDGE_executor
    PUBLIC
        WASMEDGE::common
        WASMEDGE::system
)
target_compile_features(WASMEDGE_executor
    PUBLIC
        cxx_std_17
)

# ==
# lib/wasi
# ==
add_library(WASMEDGE_hostmodulewasi)
add_library(WASMEDGE::hostmodulewasi ALIAS WASMEDGE_hostmodulewasi)
target_sources(WASMEDGE_hostmodulewasi
    PRIVATE
        lib/host/wasi/environ.cpp
        lib/host/wasi/vinode.cpp
        lib/host/wasi/wasifunc.cpp
        lib/host/wasi/wasimodule.cpp
        lib/host/wasi/vfs_io.cpp
        $<$<PLATFORM_ID:Darwin>:
            lib/host/wasi/clock-macos.cpp
            lib/host/wasi/environ-macos.cpp
            lib/host/wasi/inode-macos.cpp
            lib/host/wasi/macos.mm
        >
        $<$<PLATFORM_ID:Windows>:
            lib/host/wasi/clock-win.cpp
            lib/host/wasi/environ-win.cpp
            lib/host/wasi/inode-win.cpp
        >
        $<$<PLATFORM_ID:WINDOWS>:
            lib/host/wasi/clock-linux.cpp
            lib/host/wasi/environ-linux.cpp
            lib/host/wasi/inode-linux.cpp
        >
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
            thirdparty
        FILES
            include/host/mock/log.h
            include/host/mock/wasi_crypto_func.h
            include/host/mock/wasi_crypto_module.h
            include/host/mock/wasi_logging_func.h
            include/host/mock/wasi_logging_module.h
            include/host/mock/wasi_nn_func.h
            include/host/mock/wasi_nn_module.h
            include/host/mock/wasmedge_image_func.h
            include/host/mock/wasmedge_image_module.h
            include/host/mock/wasmedge_stablediffusion_func.h
            include/host/mock/wasmedge_stablediffusion_module.h
            include/host/mock/wasmedge_tensorflow_func.h
            include/host/mock/wasmedge_tensorflow_module.h
            include/host/mock/wasmedge_tensorflowlite_func.h
            include/host/mock/wasmedge_tensorflowlite_module.h
            include/host/wasi/clock.h
            include/host/wasi/environ.h
            include/host/wasi/error.h
            include/host/wasi/inode.h
            include/host/wasi/vfs_io.h
            include/host/wasi/vfs.h
            include/host/wasi/vinode.h
            include/host/wasi/wasibase.h
            include/host/wasi/wasifunc.h
            include/host/wasi/wasimodule.h
            thirdparty/wasi/api.hpp
            thirdparty/wasm/wasm.h
            thirdparty/wasm/wasm.hh
)
target_link_libraries(WASMEDGE_hostmodulewasi
    PUBLIC
        WASMEDGE::system
)
target_compile_features(WASMEDGE_hostmodulewasi
    PUBLIC
        cxx_std_17
)

# ==
# lib/vm
# ==
add_library(WASMEDGE_vm)
add_library(WASMEDGE::vm ALIAS WASMEDGE_vm)
target_sources(WASMEDGE_vm
    PRIVATE
        lib/vm/plugin_modules.cpp
        lib/vm/vm.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
            include/vm/vm.h
)
target_link_libraries(WASMEDGE_vm
    PUBLIC
        WASMEDGE::common
        WASMEDGE::system
        WASMEDGE::loader
        WASMEDGE::plugin
        WASMEDGE::validator
        WASMEDGE::executor
        WASMEDGE::hostmodulewasi
)
target_compile_features(WASMEDGE_vm
    PUBLIC
        cxx_std_17
)

# ==
# lib/driver
# ==
add_library(WASMEDGE_driver)
add_library(WASMEDGE::driver ALIAS WASMEDGE_driver)
target_sources(WASMEDGE_driver
    PRIVATE
        lib/driver/compilerTool.cpp
        lib/driver/runtimeTool.cpp
        lib/driver/fuzzTool.cpp
        lib/driver/fuzzPO.cpp
        lib/driver/uniTool.cpp
        lib/driver/validateTool.cpp
        lib/driver/parseTool.cpp
        lib/driver/instantiateTool.cpp
        lib/driver/toolConfig.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            include
        FILES
        include/driver/compiler.h
        include/driver/fuzzPO.h
        include/driver/fuzzTool.h
        include/driver/options.h
        include/driver/tool.h
        include/driver/unitool.h
)
target_link_libraries(WASMEDGE_driver
    PUBLIC
        WASMEDGE::vm

)
target_compile_features(WASMEDGE_driver
    PUBLIC
        cxx_std_17
)

# ==
# lib/api
# ==
configure_file(include/api/wasmedge/int128.h include/api/wasmedge/int128.h COPYONLY)
configure_file(include/api/wasmedge/version.h.in include/api/wasmedge/version.h)
configure_file(include/api/wasmedge/wasmedge.h include/api/wasmedge/wasmedge.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_ast.h include/api/wasmedge/wasmedge_ast.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_basic.h include/api/wasmedge/wasmedge_basic.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_compiler.h include/api/wasmedge/wasmedge_compiler.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_configure.h include/api/wasmedge/wasmedge_configure.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_context.h include/api/wasmedge/wasmedge_context.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_deprecated.h include/api/wasmedge/wasmedge_deprecated.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_execution.h include/api/wasmedge/wasmedge_execution.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_experimental.h include/api/wasmedge/wasmedge_experimental.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_instance.h include/api/wasmedge/wasmedge_instance.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_plugin.h include/api/wasmedge/wasmedge_plugin.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_tools.h include/api/wasmedge/wasmedge_tools.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_value.h include/api/wasmedge/wasmedge_value.h COPYONLY)
configure_file(include/api/wasmedge/wasmedge_vm.h include/api/wasmedge/wasmedge_vm.h COPYONLY)

configure_file(include/common/enum.inc include/api/wasmedge/enum.inc COPYONLY)
configure_file(include/common/enum_configure.h include/api/wasmedge/enum_configure.h COPYONLY)
configure_file(include/common/enum_errcode.h include/api/wasmedge/enum_errcode.h COPYONLY)
configure_file(include/common/enum_types.h include/api/wasmedge/enum_types.h COPYONLY)

add_library(WASMEDGE_api)
add_library(WASMEDGE::api ALIAS WASMEDGE_api)
target_sources(WASMEDGE_api
    PRIVATE
        lib/api/wasm_c.cpp
        lib/api/wasmedge.cpp
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS
            "${PROJECT_BINARY_DIR}/include/api"
        FILES
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/int128.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/version.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_ast.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_basic.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_compiler.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_configure.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_context.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_deprecated.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_execution.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_experimental.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_instance.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_plugin.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_tools.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_value.h"
            "${PROJECT_BINARY_DIR}/include/api/wasmedge/wasmedge_vm.h"
    PRIVATE
        FILE_SET private_headers
        TYPE HEADERS
        BASE_DIRS
            thirdparty/wasm
        FILES
            thirdparty/wasm/wasm.h
            thirdparty/wasm/wasm.hh
)
target_link_libraries(WASMEDGE_api
    PUBLIC
        WASMEDGE::vm
        WASMEDGE::driver
        $<$<BOOL:${WASMEDGE_USE_LLVM}>:WASMEDGE::aot>

)
target_compile_definitions(WASMEDGE_api
    PRIVATE
        WASMEDGE_COMPILE_LIBRARY
)
target_compile_features(WASMEDGE_api
    PUBLIC
        cxx_std_17
)
set_target_properties(WASMEDGE_api PROPERTIES
    VERSION "${WASMEDGE_CAPI_VERSION}"
    SOVERSION "${WASMEDGE_CAPI_SOVERSION}"
    OUTPUT_NAME wasmedge
)

if (WASMEDGE_USE_LLVM)
    # ==
    # lib/aot
    # ==
    add_library(WASMEDGE_aot)
    add_library(WASMEDGE::aot ALIAS WASMEDGE_aot)
    target_sources(WASMEDGE_aot
        PRIVATE
            lib/aot/blake3.cpp
            lib/aot/cache.cpp
        PUBLIC
            FILE_SET HEADERS
            BASE_DIRS
                include
            FILES
                include/aot/blake3.h
                include/aot/cache.h
                include/aot/version.h
    )
    target_include_directories(WASMEDGE_aot
        PRIVATE SYSTEM
            ${LLVM_INCLUDE_DIRS}
    )
    target_link_libraries(WASMEDGE_aot
        PUBLIC
            BLAKE3::blake3
            WASMEDGE::common
            WASMEDGE::system
    )
    target_compile_features(WASMEDGE_aot
        PUBLIC
            cxx_std_17
    )
    # ==
    # lib/llvm
    # ==
    add_library(WASMEDGE_llvm)
    add_library(WASMEDGE::llvm ALIAS WASMEDGE_llvm)
    target_sources(WASMEDGE_llvm
        PRIVATE
            lib/llvm/compiler.cpp
            lib/llvm/compiler/context.cpp
            lib/llvm/compiler/function_compiler.cpp
            lib/llvm/compiler/memoryInstr.cpp
            lib/llvm/compiler/numericInstr.cpp
            lib/llvm/compiler/refInstr.cpp
            lib/llvm/compiler/threadInstr.cpp
            lib/llvm/compiler/vectorInstr.cpp
            lib/llvm/codegen.cpp
            lib/llvm/data.cpp
            lib/llvm/jit.cpp
            lib/llvm/lazyjit.cpp
            lib/llvm/llvm.cpp
        PUBLIC
            FILE_SET HEADERS
            BASE_DIRS
                include
            FILES
                include/llvm/codegen.h
                include/llvm/compiler.h
                include/llvm/data.h
                include/llvm/jit.h
                include/llvm/lazyjit.h
    )
    target_include_directories(WASMEDGE_llvm
        PRIVATE SYSTEM
            ${LLVM_INCLUDE_DIRS}
    )
    target_link_libraries(WASMEDGE_llvm
        PUBLIC
            WASMEDGE::common
            WASMEDGE::system
    )
    target_compile_features(WASMEDGE_llvm
        PUBLIC
            cxx_std_17
    )
endif()

