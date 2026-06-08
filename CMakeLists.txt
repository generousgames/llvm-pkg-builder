cmake_minimum_required(VERSION 3.27)

# Name this meta-project anything; GLFW defines its own project() inside.
project(llvm-pkg-builder LANGUAGES C CXX)

########################################################
# Prebuild specific configuration.

# Prebuild utils directory.
set(MIMI_PKG_CMAKE_DIR "${CMAKE_SOURCE_DIR}/node_modules/@generousgames/mimi-pkg/cmake")

# Setup environment variables.
include("${MIMI_PKG_CMAKE_DIR}/env.cmake")

# Detect and write ABI information.
build_write_abi_json("${CMAKE_BINARY_DIR}")
message(STATUS "BUILD_TRIPLE = ${BUILD_TRIPLE}")
message(STATUS "ABI JSON : ${BUILD_ABI_JSON_PATH}")
message(STATUS "ABI HASH : ${BUILD_ABI_HASH}")

# Setup build configuration based on the manifest.
set(MANIFEST_FILE "${CMAKE_SOURCE_DIR}/manifest.json")
include("${MIMI_PKG_CMAKE_DIR}/patch.cmake")

########################################################
# Target library build configuration.

# Allow passing shared/common options from presets/CLI:
# (Your presets already set these as cache variables.)
option(BUILD_SHARED_LIBS "Build shared libs" OFF)

# If you want shared CMake helpers from your submodule:
# set(CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/third_party/prebuild-core/cmake/Modules;${CMAKE_MODULE_PATH}")

# Helpful default for prebuilts
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# Where LLVM source lives (submodule path). Allow override via -DLLVM_SOURCE_DIR=...
set(LLVM_SOURCE_DIR "${CMAKE_SOURCE_DIR}/dependencies/llvm/llvm" CACHE PATH "Path to LLVM source")

# Nice error if submodule isn't checked out.
if(NOT EXISTS "${LLVM_SOURCE_DIR}/CMakeLists.txt")
    message(FATAL_ERROR
        "LLVM not found at '${LLVM_SOURCE_DIR}'. "
        "Did you init submodules? Try: git submodule update --init --recursive")
endif()

# Forward options to the subproject BEFORE adding it, so its CMake picks them up.
set(BUILD_SHARED_LIBS ${BUILD_SHARED_LIBS} CACHE BOOL "" FORCE)

# libclang is special-cased by LLVM's build: on Windows it is produced as a DLL
# even when BUILD_SHARED_LIBS=OFF, leaving only an import lib (libclang.lib) in
# lib/ and a libclang.dll in bin/ that our libs-only bundle never ships. Force a
# genuine static libclang on every platform so consumers (e.g. mimi_reflect) link
# it in with no runtime DLL — matching how macOS already produced libclang.a.
# NOTE: consumers including <clang-c/*.h> against a static libclang must define
# CINDEX_NO_EXPORTS so the headers don't decorate the C API with __declspec(dllimport).
set(LIBCLANG_BUILD_STATIC ON CACHE BOOL "" FORCE)

# Same story for LLVM's C API: on Windows LLVM builds LLVM-C.dll by default
# (LLVM_BUILD_LLVM_C_DYLIB ON), leaving an LLVM-C.lib import lib in the bundle and
# a runtime DLL dependency. macOS never builds it. Turn it off so the Windows
# bundle is fully static and matches macOS (mimi_reflect uses libclang, not LLVM-C).
set(LLVM_BUILD_LLVM_C_DYLIB OFF CACHE BOOL "" FORCE)

# Add LLVM as a subdir; it defines targets like 'llvm'/'llvm-config'
add_subdirectory("${LLVM_SOURCE_DIR}" "${CMAKE_BINARY_DIR}/llvm")

########################################################


# If you want to expose an ALIAS with a stable name for packaging:
# GLFW uses 'glfw' (static) or 'glfw'/'glfw_shared' depending on version/options.
# Create a consistent alias to whatever exists:
# if(TARGET glfw)
#     add_library(mimi_glfw ALIAS glfw)
# elseif(TARGET glfw_shared) # some forks/configs
#     add_library(mimi_glfw ALIAS glfw_shared)
# elseif(TARGET glfw3) # older naming
#     add_library(mimi_glfw ALIAS glfw3)
# endif()

# Nothing else required: `cmake --install` will run GLFW's own install() rules.