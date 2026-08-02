cmake_minimum_required(VERSION 3.19)

# This extension only registers editor-only classes (EditorVCSInterface and
# friends), so it is built exclusively for the "editor" godot-cpp target,
# never template_debug/template_release export templates.
if(NOT DEFINED GODOTCPP_TARGET)
    set(GODOTCPP_TARGET "editor" CACHE STRING "" FORCE)
endif()

# Target the oldest Godot API godot-cpp still supports (see its README) to
# maximize which editor versions this plugin can load in. Every API surface
# used here (EditorVCSInterface's status/diff/stage/commit/branch/push/pull
# virtuals, FileAccess) is present as of 4.3; keep this in sync with
# compatibility_minimum in the .gdextension file.
if(NOT DEFINED GODOTCPP_API_VERSION)
    set(GODOTCPP_API_VERSION "4.3" CACHE STRING "" FORCE)
endif()

project(godot-lore-plugin LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

add_subdirectory(third_party/godot-cpp)

# Lore C API, built from the third_party/lore submodule via cargo. Linked
# dynamically: the staticlib archive cbindgen/cargo also produce is 100s of MB
# to several GB unlinked (no dead-code elimination happens until final link),
# whereas the cdylib is already linked+stripped down to ~30MB. lore.dll ships
# alongside the extension's own DLL at runtime.
find_program(CARGO_EXECUTABLE cargo REQUIRED)

set(LORE_SRC_DIR "${CMAKE_SOURCE_DIR}/third_party/lore")
set(LORE_CARGO_OUT_DIR "${LORE_SRC_DIR}/target/release-lto")
set(LORE_STAGE_DIR "${CMAKE_BINARY_DIR}/lore")

add_custom_target(
    lore_cargo_build
    COMMAND ${CARGO_EXECUTABLE} build --profile release-lto -p lore --lib
    COMMAND ${CMAKE_COMMAND} -E make_directory
        "${LORE_STAGE_DIR}/include" "${LORE_STAGE_DIR}/lib" "${LORE_STAGE_DIR}/bin"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${LORE_CARGO_OUT_DIR}/lore.dll" "${LORE_STAGE_DIR}/bin/lore.dll"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${LORE_CARGO_OUT_DIR}/lore.dll.lib" "${LORE_STAGE_DIR}/lib/lore.dll.lib"
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${LORE_SRC_DIR}/lore-capi/lore.h" "${LORE_STAGE_DIR}/include/lore.h"
    WORKING_DIRECTORY "${LORE_SRC_DIR}"
    COMMENT "Building lore crate (cargo, release-lto) from third_party/lore"
    VERBATIM
)

# INTERFACE_INCLUDE_DIRECTORIES on an imported target must exist at configure
# time even though lore_cargo_build doesn't populate it until the build step.
file(MAKE_DIRECTORY "${LORE_STAGE_DIR}/include")

add_library(lore SHARED IMPORTED)
set_target_properties(
    lore
    PROPERTIES
        IMPORTED_IMPLIB "${LORE_STAGE_DIR}/lib/lore.dll.lib"
        IMPORTED_LOCATION "${LORE_STAGE_DIR}/bin/lore.dll"
        INTERFACE_INCLUDE_DIRECTORIES "${LORE_STAGE_DIR}/include"
)
add_dependencies(lore lore_cargo_build)

# Thin C++ wrapper adapting lore-capi's async/event-callback API to the
# synchronous calls EditorVCSInterface's virtual methods need. Its own
# library so both the extension and the standalone FFI test binary below can
# link it without depending on godot-cpp.
add_library(lore_ffi STATIC
    src/lore_ffi/lore_c_api.h
    src/lore_ffi/lore_call.h
    src/lore_ffi/lore_client.h
    src/lore_ffi/lore_client.cpp
)
target_include_directories(lore_ffi PUBLIC src)
target_link_libraries(lore_ffi PUBLIC lore)

add_library(godot-lore-plugin SHARED
    src/register_types.cpp
    src/register_types.h
    src/lore_vcs_plugin.cpp
    src/lore_vcs_plugin.h
)

target_link_libraries(godot-lore-plugin PRIVATE godot-cpp lore_ffi)

option(GODOT_LORE_PLUGIN_BUILD_FFI_TEST "Build the standalone lore_ffi test binary (no Godot involved)" ON)
if(GODOT_LORE_PLUGIN_BUILD_FFI_TEST)
    add_executable(lore_ffi_test tests/lore_ffi_test.cpp)
    target_link_libraries(lore_ffi_test PRIVATE lore_ffi)
    set_target_properties(lore_ffi_test PROPERTIES RUNTIME_OUTPUT_DIRECTORY "$<1:${CMAKE_BINARY_DIR}/bin>")
    add_custom_command(
        TARGET lore_ffi_test POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different
            "${LORE_STAGE_DIR}/bin/lore.dll"
            "${CMAKE_BINARY_DIR}/bin/lore.dll"
    )
endif()

get_target_property(GODOTCPP_SUFFIX godot-cpp GODOTCPP_SUFFIX)

set(GODOT_LORE_PLUGIN_OUTPUT_DIR "${CMAKE_SOURCE_DIR}/addons/godot-lore-plugin/bin")

# gersemi: off
set_target_properties(
    godot-lore-plugin
    PROPERTIES
        POSITION_INDEPENDENT_CODE ON
        BUILD_RPATH_USE_ORIGIN ON

        LINK_SEARCH_START_STATIC ON
        LINK_SEARCH_END_STATIC ON

        LIBRARY_OUTPUT_DIRECTORY "$<1:${GODOT_LORE_PLUGIN_OUTPUT_DIR}>"
        RUNTIME_OUTPUT_DIRECTORY "$<1:${GODOT_LORE_PLUGIN_OUTPUT_DIR}>"
        PDB_OUTPUT_DIRECTORY     "$<1:${GODOT_LORE_PLUGIN_OUTPUT_DIR}>"

        PREFIX ""
        OUTPUT_NAME "godot-lore-plugin${GODOTCPP_SUFFIX}"
)
# gersemi: on

# lore.dll is a runtime dependency of godot-lore-plugin's own DLL; it must sit
# next to it for the loader to find it.
add_custom_command(
    TARGET godot-lore-plugin POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${LORE_STAGE_DIR}/bin/lore.dll"
        "${GODOT_LORE_PLUGIN_OUTPUT_DIR}/lore.dll"
)

# These live at the repo root (shared with the FFI test/build tooling around
# them), but the addon folder is what actually ships, so mirror them in
# alongside it. Copies, not the source of truth — gitignored.
set(GODOT_LORE_PLUGIN_ADDON_DIR "${CMAKE_SOURCE_DIR}/addons/godot-lore-plugin")
add_custom_command(
    TARGET godot-lore-plugin POST_BUILD
    COMMAND ${CMAKE_COMMAND} -E copy_if_different
        "${CMAKE_SOURCE_DIR}/THIRDPARTY.md"
        "${CMAKE_SOURCE_DIR}/LICENSE"
        "${GODOT_LORE_PLUGIN_ADDON_DIR}"
)
