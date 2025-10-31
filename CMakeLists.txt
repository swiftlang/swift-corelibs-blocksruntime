cmake_minimum_required(VERSION 3.26...3.29)

list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules)

#
# Core Project setup
#

project(libclosure
  VERSION 1.3
  LANGUAGES C CXX)

set(CMAKE_POSITION_INDEPENDENT_CODE YES)

set(CMAKE_C_STANDARD 11)
set(CMAKE_C_STANDARD_REQUIRED YES)

set(CMAKE_CXX_STANDARD 11)

#set(CMAKE_C_VISIBILITY_PRESET hidden)
#set(CMAKE_C_VISIBILITY_INLINES_HIDDEN YES)

#
# Configuration Tests
#

include(CheckCCompilerFlag)
include(CheckIncludeFiles)
include(CheckLinkerFlag)
include(CheckSymbolExists)
include(GNUInstallDirs)

# CMake standard convention for shared library support
option(BUILD_SHARED_LIBS "build shared libraries" ON)

check_c_compiler_flag("-fmodule-map-file=${CMAKE_CURRENT_SOURCE_DIR}/module.modulemap" HAVE_FMODULE_MAP_FILE)

check_linker_flag(C "LINKER:--build-id=sha1" LINKER_SUPPORTS_BUILD_ID)

check_include_files("objc/objc-internal.h" HAVE_OBJC)

check_symbol_exists(__GNU_LIBRARY__ "features.h" _GNU_SOURCE)
if(_GNU_SOURCE)
  set(CMAKE_REQUIRED_DEFINITIONS ${CMAKE_REQUIRED_DEFINITIONS} -D_GNU_SOURCE)
endif()

if(ANDROID AND "${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "aarch64")
  add_link_options("LINKER:-z,max-page-size=16384")
endif()

#
# Define the BlocksRuntime library
#

add_library(BlocksRuntime
  src/BlocksRuntime/data.c
  src/BlocksRuntime/runtime.c)

if(WIN32)
  if (NOT BUILD_SHARED_LIBS)
    target_compile_definitions(BlocksRuntime PUBLIC BlocksRuntime_STATIC)
    target_compile_options(BlocksRuntime PUBLIC
      "$<$<COMPILE_LANGUAGE:C,CXX>:SHELL:$<$<BOOL:${MSVC}>:-Xclang >-static-libclosure>"
      $<$<COMPILE_LANGUAGE:Swift>:SHELL:-Xcc -static-libclosure>)
  endif()
endif()

if (HAVE_OBJC)
  target_compile_definitions(BlocksRuntime PRIVATE HAVE_OBJC)
endif()

# Include a module map file if supported
if (HAVE_FMODULE_MAP_FILE)
  target_compile_options(BlocksRuntime PRIVATE
      "$<$<COMPILE_LANGUAGE:C,CXX>:-fmodule-map-file=${PROJECT_SOURCE_DIR}/src/BlocksRuntime/module.modulemap>")
endif()

if(LINKER_SUPPORTS_BUILD_ID)
  target_link_options(BlocksRuntime PRIVATE "LINKER:--build-id=sha1")
endif()

target_sources(BlocksRuntime
    PUBLIC
        FILE_SET HEADERS
        BASE_DIRS src
        FILES
          src/BlocksRuntime/module.modulemap
          src/BlocksRuntime/Block.h
          src/BlocksRuntime/Block_private.h)

install(TARGETS BlocksRuntime
  EXPORT BlocksRuntimeTargets
  FILE_SET HEADERS)

# Inter-project install info
export(EXPORT BlocksRuntimeTargets
       NAMESPACE BlocksRuntime::
       FILE cmake/BlocksRuntime/BlocksRuntimeTargets.cmake)
install(EXPORT BlocksRuntimeTargets
        NAMESPACE BlocksRuntime::
        DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/BlocksRuntime"
        FILE "BlocksRuntimeTargets.cmake")
include(CMakePackageConfigHelpers)
configure_package_config_file("${CMAKE_CURRENT_SOURCE_DIR}/cmake/BlocksRuntimeConfig.cmake.in"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/BlocksRuntime/BlocksRuntimeConfig.cmake"
  INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/BlocksRuntime")
write_basic_package_version_file("${CMAKE_CURRENT_BINARY_DIR}/cmake/BlocksRuntime/BlocksRuntimeConfigVersion.cmake"
  VERSION "${PROJECT_VERSION}"
  COMPATIBILITY ExactVersion)
install(FILES
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/BlocksRuntime/BlocksRuntimeConfig.cmake"
  "${CMAKE_CURRENT_BINARY_DIR}/cmake/BlocksRuntime/BlocksRuntimeConfigVersion.cmake"
  DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/BlocksRuntime")
