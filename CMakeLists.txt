# #############################################################################
# Copyright (C) 2020 - 2022 Advanced Micro Devices, Inc. All rights reserved.
#
# Permission is hereby granted, free of charge, to any person obtaining a copy
# of this software and associated documentation files (the "Software"), to deal
# in the Software without restriction, including without limitation the rights
# to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
# copies of the Software, and to permit persons to whom the Software is
# furnished to do so, subject to the following conditions:
#
# The above copyright notice and this permission notice shall be included in
# all copies or substantial portions of the Software.
#
# THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
# IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
# FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT.  IN NO EVENT SHALL THE
# AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
# LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
# OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN
# THE SOFTWARE.
# ############################################################################

# CMake version according to latest ROCm platform requirements
cmake_minimum_required( VERSION 3.16 )

# We use C++17 features, this will add compile option: -std=c++17
set( CMAKE_CXX_STANDARD 17 )
set(CMAKE_CXX_EXTENSIONS OFF)

# Consider removing this in the future
# This should appear before the project command, because it does not use FORCE
if( WIN32 )
  set( CMAKE_INSTALL_PREFIX "${PROJECT_BINARY_DIR}/package" CACHE PATH
    "Install path prefix, prepended onto install directories" )
else()
  set( CMAKE_INSTALL_PREFIX "/opt/rocm" CACHE PATH
    "Install path prefix, prepended onto install directories" )
endif()


# Workarounds..
list( APPEND CMAKE_PREFIX_PATH /opt/rocm/llvm /opt/rocm )
list( APPEND CMAKE_MODULE_PATH ${ROCM_PATH}/lib/cmake/hip /opt/rocm/lib/cmake/hip /opt/rocm/hip/cmake )

# This has to be initialized before the project() command appears
# Set the default of CMAKE_BUILD_TYPE to be release, unless user specifies with -D.
# MSVC_IDE does not use CMAKE_BUILD_TYPE
if( NOT DEFINED CMAKE_CONFIGURATION_TYPES AND NOT DEFINED CMAKE_BUILD_TYPE )
  set( CMAKE_BUILD_TYPE Release CACHE STRING "Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel." )
endif()

set( HIPFFT_BUILD_SCOPE ON )

project( hipfft LANGUAGES CXX )

# Build options
option( BUILD_SHARED_LIBS "Build ${PROJECT_NAME} as a shared library" ON )
option( BUILD_VERBOSE "Output additional build information" OFF )

set( BUILD_WITH_COMPILER "HOST-default" CACHE INTERNAL
     "Build ${PROJECT_NAME} with compiler HIP-clang, HIP-nvcc, or just the host default compiler, eg g++")
set( BUILD_WITH_LIB "ROCM" CACHE STRING "Build ${PROJECT_NAME} with ROCM or CUDA libraries" )

option( BUILD_CLIENTS "Build all clients" OFF)
option( BUILD_CLIENTS_BENCH "Build benchmark client" OFF )
option( BUILD_CLIENTS_TESTS "Build ${PROJECT_NAME} tests (requires 3rd dependencies)" OFF )
option( BUILD_CLIENTS_SAMPLES "Build examples" OFF )
option(BUILD_ADDRESS_SANITIZER "Build with address sanitizer enabled" OFF)

option( WERROR "Treat warnings as errors" OFF )

# Set internal BUILD_WITH_COMPILER.
if(NOT (CMAKE_CXX_COMPILER MATCHES ".*hipcc$" OR CMAKE_CXX_COMPILER MATCHES ".*clang\\+\\+"))
  set( BUILD_WITH_COMPILER "HOST-default" )
else()
  if( $ENV{HIP_PLATFORM} MATCHES "nvidia" )
    set( BUILD_WITH_COMPILER "HIP-nvcc" )
  else()
    set( BUILD_WITH_COMPILER "HIP-clang" )
    if( NOT BUILD_WITH_LIB STREQUAL "ROCM" )
      message( FATAL_ERROR "Detected HIP_COMPILER=clang, but BUILD_WITH_LIB is not ROCM!" )
    endif()
  endif()
endif()

string( TOUPPER "${BUILD_WITH_COMPILER}" BUILD_WITH_COMPILER )
string( TOUPPER "${BUILD_WITH_LIB}" BUILD_WITH_LIB )

set( WARNING_FLAGS -Wall -Wno-unused-function -Wimplicit-fallthrough -Wunreachable-code -Wno-unknown-pragmas)
if( WERROR )
  set( WARNING_FLAGS ${WARNING_FLAGS} -Werror )
endif()

if (BUILD_WITH_COMPILER STREQUAL "HIP-NVCC" )
  set (BUILD_WITH_LIB "CUDA")
  set( HIP_PLATFORM "nvidia" )

  set( CMAKE_CXX_EXTENSIONS OFF )

  set( CMAKE_CXX_COMPILE_OPTIONS_PIC "-Xcompiler=${CMAKE_CXX_COMPILE_OPTIONS_PIC}" )

  set( CMAKE_SHARED_LIBRARY_C_FLAGS "-Xlinker=${CMAKE_SHARED_LIBRARY_C_FLAGS}" )
  set( CMAKE_SHARED_LIBRARY_CXX_FLAGS "-Xlinker=${CMAKE_SHARED_LIBRARY_CXX_FLAGS}" )
  set( CMAKE_SHARED_LIBRARY_SONAME_C_FLAG "-Xlinker=-soname," )
  set( CMAKE_SHARED_LIBRARY_SONAME_CXX_FLAG "-Xlinker=-soname," )

  set( CMAKE_SHARED_LIBRARY_RUNTIME_C_FLAG "-Xlinker=-rpath," )
  set( CMAKE_SHARED_LIBRARY_RUNTIME_CXX_FLAG "-Xlinker=-rpath," )
  set( CMAKE_EXECUTABLE_RUNTIME_C_FLAG "-Xlinker=-rpath," )
  set( CMAKE_EXECUTABLE_RUNTIME_CXX_FLAG "-Xlinker=-rpath," )

  set( CMAKE_C_COMPILE_OPTIONS_VISIBILITY "-Xcompiler='${CMAKE_C_COMPILE_OPTIONS_VISIBILITY}'" )
  set( CMAKE_CXX_COMPILE_OPTIONS_VISIBILITY "-Xcompiler='${CMAKE_CXX_COMPILE_OPTIONS_VISIBILITY}'" )
  set( CMAKE_C_COMPILE_OPTIONS_VISIBILITY_INLINES_HIDDEN
    "-Xcompiler='${CMAKE_C_COMPILE_OPTIONS_VISIBILITY_INLINES_HIDDEN}'" )
  set( CMAKE_CXX_COMPILE_OPTIONS_VISIBILITY_INLINES_HIDDEN
    "-Xcompiler='${CMAKE_CXX_COMPILE_OPTIONS_VISIBILITY_INLINES_HIDDEN}'" )

  foreach( FLAG IN ITEMS ${WARNING_FLAGS} )
    set( NVCC_WARNING_FLAGS ${NVCC_WARNING_FLAGS} "-Xcompiler=${FLAG}" )
  endforeach()
  set( WARNING_FLAGS ${NVCC_WARNING_FLAGS} )

  else()
  # Define GPU targets
  set( AMDGPU_TARGETS gfx803;gfx900;gfx906;gfx908;gfx90a;gfx940;gfx941;gfx942;gfx1030;gfx1100;gfx1101;gfx1102;gfx1151;gfx1200;gfx1201 CACHE STRING
  "List of specific machine types for library to target" )

  if( BUILD_WITH_COMPILER STREQUAL "HIP-CLANG" )
    set( HIP_PLATFORM "amd" )
    set( HIP_COMPILER "clang" )
    if(BUILD_ADDRESS_SANITIZER)
      set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address -shared-libasan")
      set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fsanitize=address -shared-libasan")
      add_link_options(-fuse-ld=lld)
    endif()
  endif()
endif()

# Show the actual compiler(internal option)
message(STATUS "BUILD_WITH_COMPILER = " ${BUILD_WITH_COMPILER})

# Dependencies
include(cmake/dependencies.cmake)

# FOR HANDLING ENABLE/DISABLE OPTIONAL BACKWARD COMPATIBILITY for FILE/FOLDER REORG
option(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY "Build with file/folder reorg with backward compatibility enabled" OFF)
if(BUILD_FILE_REORG_BACKWARD_COMPATIBILITY AND NOT WIN32)
  rocm_wrap_header_dir(
    ${CMAKE_SOURCE_DIR}/library/include
    PATTERNS "*.h"
    GUARDS SYMLINK WRAPPER
    WRAPPER_LOCATIONS ${CMAKE_INSTALL_INCLUDEDIR}
  )
endif()

# Version
set( VERSION_STRING "1.0.17" )
set( hipfft_SOVERSION 0.1 )

if( ROCM_FOUND )
  rocm_setup_version( VERSION ${VERSION_STRING} )
endif()

add_subdirectory( library )

# Build clients of the library
if( BUILD_CLIENTS )
  set( BUILD_CLIENTS_BENCH ON )
  set( BUILD_CLIENTS_SAMPLES ON )
  set( BUILD_CLIENTS_TESTS ON )
endif()

# old name for BUILD_CLIENTS_BENCH
if( BUILD_CLIENTS_RIDER )
  set( BUILD_CLIENTS_BENCH ${BUILD_CLIENTS_RIDER} )
endif()

# Build clients of the library
if( BUILD_CLIENTS_BENCH OR BUILD_CLIENTS_SAMPLES OR BUILD_CLIENTS_TESTS )
  include( clients/cmake/build-options.cmake )
  rocm_package_setup_component(clients)
  if(NOT CLIENTS_OS)
    rocm_set_os_id(CLIENTS_OS)
    string(TOLOWER "${CLIENTS_OS}" CLIENTS_OS)
    rocm_read_os_release(CLIENTS_OS_VERSION VERSION_ID)
  endif()
  message(STATUS "OS: ${CLIENTS_OS} ${CLIENTS_OS_VERSION}")
  set(FFTW_DEB "libfftw3-bin")
  if(CLIENTS_OS STREQUAL "sles")
    set(FFTW_RPM "libfftw3-3")
  elseif(CLIENTS_OS STREQUAL "mariner")
    set(BOOST_RPM RPM "boost = ${Boost_VERSION_MAJOR}_${Boost_VERSION_MINOR}_${Boost_VERSION_PATCH}")
    set(FFTW_RPM "fftw-libs")
  else()
    set(FFTW_RPM "fftw-libs")
  endif()
  if(BUILD_CLIENTS_TESTS)
    rocm_package_setup_client_component(
      tests
      DEPENDS
        DEB ${FFTW_DEB} rocrand
        RPM ${FFTW_RPM} rocrand
    )
  endif()
  if(BUILD_CLIENTS_BENCH)
    rocm_package_setup_client_component(
      benchmarks
      DEPENDS
        DEB rocrand
        RPM rocrand
    )
  endif()
  add_subdirectory( clients )
endif()

# Packaging...
if(WIN32)
  set(CPACK_SOURCE_GENERATOR "ZIP")
  set(CPACK_GENERATOR "ZIP")
  set(CMAKE_INSTALL_PREFIX "C:/hipSDK" CACHE PATH "Install path" FORCE)
  set(INSTALL_PREFIX "C:/hipSDK")
  set(CPACK_SET_DESTDIR OFF)
  set(CPACK_PACKAGE_INSTALL_DIRECTORY "C:/hipSDK")
  set(CPACK_PACKAGING_INSTALL_PREFIX "")
  set(CPACK_INCLUDE_TOPLEVEL_DIRECTORY OFF)
endif()

if( ROCM_FOUND )
  # Package specific CPACK vars
  if( NOT BUILD_WITH_LIB STREQUAL "CUDA" )
    rocm_package_add_dependencies(DEPENDS "rocfft >= 1.0.21")
  else()
    rocm_package_add_dependencies(DEPENDS "cufft >= 10.0.0")
  endif()

  set( CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.md" )

  set( CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "\${CPACK_PACKAGING_INSTALL_PREFIX}" )

  # Give hipfft compiled for CUDA backend a different name
  if( BUILD_WITH_LIB STREQUAL "ROCM" )
    set( package_name hipfft )
  else()
    set( package_name hipfft-alt )
  endif()

  set( HIPFFT_CONFIG_DIR "\${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_LIBDIR}" CACHE PATH "Path placed into ldconfig file" )

  rocm_create_package(
    NAME ${package_name}
    DESCRIPTION "ROCm FFT marshalling library"
    MAINTAINER "hipfft-maintainer@amd.com"
    LDCONFIG
    LDCONFIG_DIR ${HIPFFT_CONFIG_DIR}
    )
endif()
