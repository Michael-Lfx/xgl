##
 #######################################################################################################################
 #
 #  Copyright (c) 2017-2020 Advanced Micro Devices, Inc. All Rights Reserved.
 #
 #  Permission is hereby granted, free of charge, to any person obtaining a copy
 #  of this software and associated documentation files (the "Software"), to deal
 #  in the Software without restriction, including without limitation the rights
 #  to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
 #  copies of the Software, and to permit persons to whom the Software is
 #  furnished to do so, subject to the following conditions:
 #
 #  The above copyright notice and this permission notice shall be included in all
 #  copies or substantial portions of the Software.
 #
 #  THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
 #  IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
 #  FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
 #  AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
 #  LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
 #  OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
 #  SOFTWARE.
 #
 #######################################################################################################################

cmake_minimum_required(VERSION 3.5)
project(XGL VERSION 1 LANGUAGES C CXX)

# Before GCC7, when LTO is enabled, undefined reference error was observed when linking static libraries.
# Use the gcc-ar wrapper instead of ar, this invokes ar with the right plugin arguments
# --plugin /usr/lib/gcc/.../liblto_plugin.so
if(UNIX)
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        find_program(GCC_AR gcc-ar)
        if (GCC_AR)
             set(CMAKE_AR ${GCC_AR})
        else()
            message(FATAL_ERROR "gcc-ar cannot be found!")
        endif()
        find_program(GCC_RANLIB gcc-ranlib)
        if (GCC_RANLIB)
            set(CMAKE_RANLIB ${GCC_RANLIB})
        else()
            message(FATAL_ERROR "gcc-ranlib cannot be found!")
        endif()
    endif()
endif()

### Set Default Build Type #############################################################################################
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  message(STATUS "No build type selected, default to release")
  set(CMAKE_BUILD_TYPE "Release" CACHE STRING "Build type (default Release)" FORCE)
endif()

### CMake Includes #####################################################################################################
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/../pal/cmake/Modules)
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} ${PROJECT_SOURCE_DIR}/icd/imported/pal/cmake/Modules)
set(GLOBAL_ROOT_SRC_DIR ${PROJECT_SOURCE_DIR}/../../ CACHE PATH "" FORCE)

# Generate a CompilationDatabase (compile_commands.json file) for our build
# for intellisense used by VScode, YouCompleteMe, etc
set(CMAKE_EXPORT_COMPILE_COMMANDS 1)

include(CMakeDependentOption)
include(AmdCmakeHelper)
include(SetupAmdGlobalRoots)

### Strip the symbols for release build ################################################################################
if(UNIX)
    if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
        string(APPEND CMAKE_SHARED_LINKER_FLAGS_RELEASE " -s")
    endif()
endif()

### Cached Project Options #############################################################################################

option(ICD_BUILD_LLPC "Build LLPC?" ON)
option(XGL_BUILD_VEGA20 "Build open source vulkan for Vega20?" ON)
option(XGL_BUILD_RAVEN2 "Build open source vulkan for Raven2?" ON)
option(XGL_BUILD_RENOIR "Build open source vulkan for Renoir?" ON)

option(XGL_BUILD_LIT "Build with Lit test?" OFF)
option(XGL_BUILD_GFX10 "Build open source vulkan for GFX10" ON)

cmake_dependent_option(XGL_BUILD_NAVI14 "Build open source vulkan for Navi14" ON "XGL_BUILD_GFX10" OFF)

#if VKI_KHR_SHADER_SUBGROUP_EXTENDED_TYPES
option(VKI_KHR_SHADER_SUBGROUP_EXTENDED_TYPES "Build vulkan with KHR_SHADER_SUBGROUP_EXTENDED_TYPES" OFF)
#endif
option(ICD_BUILD_LLPCONLY "Build LLPC Only?" OFF)

if(ICD_BUILD_LLPCONLY)
    set(ICD_BUILD_LLPC ON CACHE BOOL "ICD_BUILD_LLPCONLY override." FORCE)
endif()

option(ICD_BUILD_VIDEO "Build Video Support?" OFF)

option(XGL_LLVM_UPSTREAM "Build with upstreamed LLVM?" OFF)

if(NOT ICD_BUILD_LLPC)
    set(XGL_LLVM_UPSTREAM OFF CACHE BOOL "XGL_LLVM_UPSTREAM is overrided to false." FORCE)
endif()

option(ICD_GPUOPEN_DEVMODE_BUILD "Build ${PROJECT_NAME} with GPU Open Developer Mode driver support?" ON)

option(ICD_MEMTRACK "Turn on memory tracking?" ${CMAKE_BUILD_TYPE_DEBUG})

option(BUILD_WAYLAND_SUPPORT "Build XGL with Wayland support" ON)

option(BUILD_XLIB_XRANDR_SUPPORT "Build Xlib with xrandr 1.6 support" OFF)

option(ICD_ANALYSIS_WARNINGS_AS_ERRORS "Warnings as errors?" OFF)

# Option Overrides #################################################################################
# These are options that override XGL subproject options.  As these overrides are managed and force
# set by XGL, mark_as_advanced is used to hide them from the CMake GUI.

# This will become the value of PAL_CLIENT_INTERFACE_MAJOR_VERSION.  It describes the version of the PAL interface
# that the ICD supports.  PAL uses this value to enable backwards-compatibility for older interface versions.  It must
# be updated on each PAL promotion after handling all of the interface changes described in palLib.h.
file(STRINGS icd/make/importdefs PAL_MAJOR_VERSION REGEX "^ICD_PAL_CLIENT_MAJOR_VERSION = [0-9]+")

if(PAL_MAJOR_VERSION STREQUAL "")
    message(STATUS "Failed to find ICD_PAL_CLIENT_MAJOR_VERSION")
else()
    string(REGEX REPLACE "ICD_PAL_CLIENT_MAJOR_VERSION = " "" PAL_MAJOR_VERSION ${PAL_MAJOR_VERSION})
    message(STATUS "Detected ICD_PAL_CLIENT_MAJOR_VERSION is " ${PAL_MAJOR_VERSION})
endif()

set(ICD_PAL_CLIENT_MAJOR_VERSION ${PAL_MAJOR_VERSION})

# Handle MINOR_VERSION in the same way
file(STRINGS icd/make/importdefs PAL_MINOR_VERSION REGEX "^ICD_PAL_CLIENT_MINOR_VERSION = [0-9]+")

if(PAL_MINOR_VERSION STREQUAL "")
    message(STATUS "Failed to find ICD_PAL_CLIENT_MINOR_VERSION")
else()
    string(REGEX REPLACE "ICD_PAL_CLIENT_MINOR_VERSION = " "" PAL_MINOR_VERSION ${PAL_MINOR_VERSION})
    message(STATUS "Detected ICD_PAL_CLIENT_MINOR_VERSION is " ${PAL_MINOR_VERSION})
endif()

set(ICD_PAL_CLIENT_MINOR_VERSION ${PAL_MINOR_VERSION})

# This will become the value of LLPC_CLIENT_INTERFACE_MAJOR_VERSION.  It describes the version of the LLPC interface
# that the ICD supports.  LLPC uses this value to enable backwards-compatibility for older interface versions.  It must
# be updated on each LLPC promotion after handling all of the interface changes described in llpc.h
file(STRINGS icd/make/importdefs LLPC_MAJOR_VERSION REGEX "^ICD_LLPC_CLIENT_MAJOR_VERSION = [0-9]+")

if(LLPC_MAJOR_VERSION STREQUAL "")
    message(STATUS "Failed to find ICD_LLPC_CLIENT_MAJOR_VERSION")
else()
    string(REGEX REPLACE "ICD_LLPC_CLIENT_MAJOR_VERSION = " "" LLPC_MAJOR_VERSION ${LLPC_MAJOR_VERSION})
    message(STATUS "Detected ICD_LLPC_CLIENT_MAJOR_VERSION is " ${LLPC_MAJOR_VERSION})
endif()

set(ICD_LLPC_CLIENT_MAJOR_VERSION ${LLPC_MAJOR_VERSION})

# GPUOPEN
if(ICD_GPUOPEN_DEVMODE_BUILD)
    file(STRINGS icd/make/importdefs GPUOPEN_MAJOR_VERSION REGEX "^ICD_GPUOPEN_CLIENT_MAJOR_VERSION = [0-9]+")

    if(GPUOPEN_MAJOR_VERSION STREQUAL "")
        message(STATUS "Failed to find ICD_GPUOPEN_CLIENT_MAJOR_VERSION")
    else()
        string(REGEX REPLACE "ICD_GPUOPEN_CLIENT_MAJOR_VERSION = " "" GPUOPEN_MAJOR_VERSION ${GPUOPEN_MAJOR_VERSION})
        message(STATUS "Detected ICD_GPUOPEN_CLIENT_MAJOR_VERSION is " ${GPUOPEN_MAJOR_VERSION})
    endif()
    set(GPUOPEN_CLIENT_INTERFACE_MAJOR_VERSION ${GPUOPEN_MAJOR_VERSION})
endif()

# PAL
set(PAL_BUILD_JEMALLOC OFF CACHE BOOL "Force jemalloc off" FORCE)
set(XGL_PAL_PATH ${PROJECT_SOURCE_DIR}/../pal CACHE PATH "Specify the path to the PAL project.")
set(PAL_CLIENT_INTERFACE_MAJOR_VERSION ${ICD_PAL_CLIENT_MAJOR_VERSION} CACHE STRING "${PROJECT_NAME} override." FORCE)
set(PAL_CLIENT_INTERFACE_MINOR_VERSION ${ICD_PAL_CLIENT_MINOR_VERSION} CACHE STRING "${PROJECT_NAME} override." FORCE)
set(PAL_CLIENT "VULKAN" CACHE STRING "${PROJECT_NAME} override." FORCE)

set(PAL_MEMTRACK ${ICD_MEMTRACK} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_GPUOPEN ${ICD_GPUOPEN_DEVMODE_BUILD} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_VIDEO ${ICD_BUILD_VIDEO} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_RAVEN2 ${XGL_BUILD_RAVEN2} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_RENOIR ${XGL_BUILD_RENOIR} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_VEGA20 ${XGL_BUILD_VEGA20} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_GFX10 ${XGL_BUILD_GFX10} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(PAL_BUILD_NAVI14 ${XGL_BUILD_NAVI14} CACHE BOOL "${PROJECT_NAME} override." FORCE)

# Wayland
if (BUILD_WAYLAND_SUPPORT)
    set(PAL_BUILD_WAYLAND ON CACHE BOOL "Build PAL with Wayland support" FORCE)
    find_package(PkgConfig REQUIRED)
    pkg_check_modules(WAYLAND REQUIRED wayland-client)
endif()

### Generator Dependencies #############################################################################################
find_package(PythonInterp 3)
if(NOT PYTHONINTERP_FOUND)
    if(UNIX)
        message(FATAL_ERROR "Python 3 is needed to generate some source files.")
    endif()
else()
    if(UNIX)
        if(${PYTHON_VERSION_MINOR} LESS "6")
            message(FATAL_ERROR "Python 3.6(CPython) or higher is needed to generate some source files.")
        endif()
    endif()
endif()
set(PYTHON_CMD ${PYTHON_EXECUTABLE})

find_package(Perl 5)
if(NOT PERL_FOUND)
    if(UNIX)
        message(FATAL_ERROR "Perl 5 is needed to generate some source files.")
    endif()
endif()
set(PERL_CMD ${PERL_EXECUTABLE})

### Add Subdirectories #################################################################################################

### Metrohash ##########################################################
if(EXISTS ${PROJECT_SOURCE_DIR}/../MetroHash)
    set(XGL_METROHASH_PATH ${PROJECT_SOURCE_DIR}/../MetroHash CACHE PATH "The path of metrohash.")
else()
    set(XGL_METROHASH_PATH ${PROJECT_SOURCE_DIR}/../third_party/metrohash CACHE PATH "The path of metrohash.")
endif()

if(EXISTS ${XGL_METROHASH_PATH} AND NOT TARGET metrohash)
    add_subdirectory(${XGL_METROHASH_PATH} ${PROJECT_BINARY_DIR}/metrohash)
endif()

### cwpack #############################################################
if(EXISTS ${PROJECT_SOURCE_DIR}/../CWPack)
    set(XGL_CWPACK_PATH ${PROJECT_SOURCE_DIR}/../CWPack CACHE PATH "The path of cwpack.")
else()
    set(XGL_CWPACK_PATH ${PROJECT_SOURCE_DIR}/../third_party/cwpack CACHE PATH "The path of cwpack.")
endif()

if(EXISTS ${XGL_CWPACK_PATH} AND NOT TARGET cwpack)
    add_subdirectory(${XGL_CWPACK_PATH} ${PROJECT_BINARY_DIR}/cwpack)
endif()

### VKGC ###############################################################
set(XGL_ICD_PATH ${PROJECT_SOURCE_DIR}/icd CACHE PATH "The path of xgl, it is read-only.")

if (EXISTS ${XGL_ICD_PATH}/api/compiler/CMakeLists.txt)
    set(XGL_VKGC_PATH ${XGL_ICD_PATH}/api/compiler CACHE PATH "Specify the path to the compiler." FORCE)
else()
    # On github, the default repo name is llpc instead of compiler
    set(XGL_VKGC_PATH ${PROJECT_SOURCE_DIR}/../llpc CACHE PATH "Specify the path to the llpc repository." FORCE)
endif()

set(LLPC_CLIENT_INTERFACE_MAJOR_VERSION ${ICD_LLPC_CLIENT_MAJOR_VERSION} CACHE STRING "${PROJECT_NAME} override." FORCE)
set(LLPC_BUILD_LIT ${XGL_BUILD_LIT} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(LLPC_BUILD_GFX10 ${XGL_BUILD_GFX10} CACHE BOOL "${PROJECT_NAME} override." FORCE)
set(LLPC_BUILD_NAVI14 ${XGL_BUILD_NAVI14} CACHE BOOL "${PROJECT_NAME} override." FORCE)
if(ICD_BUILD_LLPC)
    set(LLPC_BUILD_RAVEN2 ${XGL_BUILD_RAVEN2} CACHE BOOL "${PROJECT_NAME} override." FORCE)
    set(LLPC_BUILD_RENOIR ${XGL_BUILD_RENOIR} CACHE BOOL "${PROJECT_NAME} override." FORCE)
    set(LLPC_BUILD_VEGA20 ${XGL_BUILD_VEGA20} CACHE BOOL "${PROJECT_NAME} override." FORCE)
    set(LLPC_ENABLE_WERROR ${ICD_ANALYSIS_WARNINGS_AS_ERRORS} CACHE BOOL "${PROJECT_NAME} override." FORCE)
endif()

add_subdirectory(${XGL_VKGC_PATH} ${CMAKE_BINARY_DIR}/compiler)

if(NOT ICD_BUILD_LLPCONLY)
    add_subdirectory(icd)
endif()
