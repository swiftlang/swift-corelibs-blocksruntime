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

check_c_compiler_flag(-fmodule-map-file HAVE_FMODULE_MAP_FILE)

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
  target_sources(BlocksRuntime PRIVATE
    BlocksRuntime.def)

  if(NOT BUILD_SHARED_LIBS)
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
  target_compile_options(BlocksRuntime
      $<$<COMPILE_LANGUAGE:C,CXX>:-fmodule-map-file=${PROJECT_SOURCE_DIR}/module.modulemap>)
endif()

# TODO: How should the module map file be installed?
#install(FILES ${PROJECT_SOURCE_DIR}/module.modulemap
#        TYPE LIB)

if(LINKER_SUPPORTS_BUILD_ID)
  target_link_options(BlocksRuntime PRIVATE "LINKER:--build-id=sha1")
endif()

# Public headers to be made available to clients
set_target_properties(BlocksRuntime
               PROPERTIES
	       PUBLIC_HEADER
	        "src/BlocksRuntime/Block.h;src/BlocksRuntime/Block_private.h")

# Install the library and headers
# Headers go into BlocksRuntime subdir of the usual include location.
install(TARGETS BlocksRuntime
        EXPORT BlocksRuntime
	PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/BlocksRuntime)

# Make it possible to find_package and use this library
# from the build directory
export(TARGETS BlocksRuntime
       NAMESPACE BlocksRuntime::
       FILE cmake/export/BlocksRuntime.cmake)

# Make it possible to use find_package with this library
# after it is installed.
# FIXME: `CMAKE_INSTALL_LIBDIR` here is certainly not the right location
install(EXPORT BlocksRuntime
        DESTINATION ${CMAKE_INSTALL_LIBDIR})
