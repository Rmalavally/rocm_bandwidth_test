cmake_minimum_required(VERSION 3.15)
#
#  Setup build environment
#
#  1) Setup cmake variable CMAKE_PREFIX_PATH to point to a root
#     directory that has ROCr header and ROCr libraries
#
#     export CMAKE_PREFIX_PATH="Path to ROCr Header and ROCr libraries"
#
#       e.g. export CMAKE_PREFIX_PATH=/opt/rocm/
#       e.g. export CMAKE_PREFIX_PATH=${HOME}/git/compute/out/ubuntu-16.04/16.04/
#
#  2) Make an new folder called build under root folder
#
#     mkdir build
#
#  3) Enter into folder of build, and run CMAKE to generate makefile
#     and make it
#
#     cd build
#     cmake  ..
#     make
#
#     @note: Add -DCMAKE_BUILD_TYPE=Debug       # if you want to build Debug
#     @note: Add -DCMAKE_PREFIX_PATH="Rocm Dir" # if you don't define Env var
#
#

# Specify name of project to build, install and package
set(PROJECT_NAME "rocm-bandwidth-test")
set(TEST_NAME "${PROJECT_NAME}")
project(${PROJECT_NAME})

include(GNUInstallDirs)
# Build is not supported on Windows plaform
if(WIN32)
  message("Windows platfom is not supported")
  return()
endif()

# Flag to enable / disable verbose output.
option(CMAKE_VERBOSE_MAKEFILE "Enable verbose output" ON)
option(CMAKE_EXPORT_COMPILE_COMMANDS "Export compile commands for linters and autocompleters" ON)

# Compiler Preprocessor definitions.
add_definitions(-DUNIX_OS)
add_definitions(-DLINUX)
add_definitions(-D__AMD64__)
add_definitions(-DAMD_INTERNAL_BUILD)
add_definitions(-DLITTLEENDIAN_CPU=1)
add_definitions(-DHSA_LARGE_MODEL=)
add_definitions(-DHSA_DEPRECATED=)

# Enable debug trace
if(DEFINED ENV{CMAKE_DEBUG_TRACE})
  add_definitions(-DDEBUG_TRACE=1)
endif()

# Linux Compiler options
# Allow compiler flags to inherit any set by env
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Werror=return-type")

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fPIC")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-rtti")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fexceptions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-math-errno")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fms-extensions")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fvisibility=hidden")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fmerge-all-constants")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fno-threadsafe-statics")

# CLANG options
if("$ENV{CXX}" STREQUAL "/usr/bin/clang++")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ferror-limit=1000000")
endif()

# Use the EXE linker flags from build scripts as well
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-Bdynamic -Wl,-z,noexecstack")
set(CMAKE_SKIP_BUILD_RPATH TRUE)

## Address Sanitize Flag
if ( ${ADDRESS_SANITIZER} )
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address")
    set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -fsanitize=address")
else ()
    if (CMAKE_COMPILER_IS_GNUCC)
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-no-undefined")
    else ()
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,-undefined,error")
    endif ()
endif ()

# Check Env vars
if(NOT DEFINED CMAKE_BUILD_TYPE OR "${CMAKE_BUILD_TYPE}" STREQUAL "")
  if(DEFINED ENV{CMAKE_BUILD_TYPE})
    set(CMAKE_BUILD_TYPE $ENV{CMAKE_BUILD_TYPE})
  endif()
endif()

# Extend Compiler flags based on build type
string(TOLOWER "${CMAKE_BUILD_TYPE}" CMAKE_BUILD_TYPE)
if("${CMAKE_BUILD_TYPE}" STREQUAL debug)
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ggdb")
  set(CMAKE_BUILD_TYPE "debug")
else()
  set(CMAKE_BUILD_TYPE "release")
endif()

# Extend Compiler flags based on Processor architecture
if("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86_64")
  set(NBIT 64)
  set(NBITSTR "64")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m64  -msse -msse2")
elseif("${CMAKE_SYSTEM_PROCESSOR}" STREQUAL "x86")
  set(NBIT 32)
  set(NBITSTR "")
  set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -m32")
endif()

# Set project requirements
set(CORE_RUNTIME_NAME "hsa-runtime")
set(CORE_RUNTIME_TARGET "${CORE_RUNTIME_NAME}64")

# Add cmake_modules to default module path if it is not
# already set and include utils from cmake modules
if(NOT DEFINED CMAKE_MODULE_PATH)
  set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake_modules/")
endif()
include(utils)

# Using find_package(has-runtime64 to find required header and library files
# This scheme could fail when using older builds of ROCm. In such a case the
# build system relies on user defined locations to find header and library files
find_package(hsa-runtime64 PATHS /opt/rocm REQUIRED)
# hsa-runtime config files will provide the include path via INSTALL_INTERFACE
message("hsa-runtime64 found @  ${hsa-runtime64_DIR} ")

#
# Set the package version for the test. It is critical that this
# value track what is used in the test source. The code from utils
# module will parse the string into major, minor and patch sub-fields
#
get_version()

# Bind the Major, Minor and Patch values
set(BUILD_VERSION_MAJOR ${VERSION_MAJOR})
set(BUILD_VERSION_MINOR ${VERSION_MINOR})

# Basic Tool Chain Information
message(" ")
message("----------------NBIT: ${NBIT}")
message("-----------BuildType: ${CMAKE_BUILD_TYPE}")
message("------------Compiler: ${CMAKE_CXX_COMPILER}")
message("----Compiler-Version: ${CMAKE_CXX_COMPILER_VERSION}")
message("-----CMAKE_CXX_FLAGS: ${CMAKE_CXX_FLAGS}")
message("---CMAKE_PREFIX_PATH: ${CMAKE_PREFIX_PATH}")
message(" ")

# Add sources that belong to the project
aux_source_directory(${CMAKE_CURRENT_SOURCE_DIR} Src)

# Build and link the test program
add_executable(${TEST_NAME} ${Src})
target_link_libraries(${TEST_NAME} PRIVATE hsa-runtime64::hsa-runtime64)
target_link_libraries(${TEST_NAME} PRIVATE c stdc++ dl pthread rt)

# Update linker flags to include RPATH
# Add --enable-new-dtags to generate DT_RUNPATH
if(DEFINED ENV{ROCM_RPATH})
   set(CMAKE_EXE_LINKER_FLAGS "-Wl,--enable-new-dtags -Wl,--rpath,$ENV{ROCM_RPATH}")
endif()

# Add install directives for rocm_bandwidth_test
install(TARGETS ${TEST_NAME} RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})

# Add packaging directives for rocm_bandwidth_test
set(CPACK_PACKAGE_NAME ${PROJECT_NAME})
set(CPACK_PACKAGE_VENDOR "Advanced Micro Devices, Inc.")
set(CPACK_PACKAGE_VERSION_MAJOR ${BUILD_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${BUILD_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${BUILD_VERSION_PATCH})
set(CPACK_PACKAGE_CONTACT "TODO <Add a valid email id>")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "Diagnostic utility tool to measure PCIe bandwidth on ROCm platforms")

# Make proper version for appending
# Default Value is 99999, setting it first
set(ROCM_VERSION_FOR_PACKAGE "99999")
if(DEFINED ENV{ROCM_LIBPATCH_VERSION})
  set(ROCM_VERSION_FOR_PACKAGE $ENV{ROCM_LIBPATCH_VERSION})
endif()

#Set the License information
set( CPACK_RPM_PACKAGE_LICENSE "NCSA" )
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/LICENSE.txt")
install(FILES ${CPACK_RESOURCE_FILE_LICENSE} DESTINATION ${CMAKE_INSTALL_DOCDIR})

# Debian package specific variables
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libstdc++6, hsa-rocr")
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://github.com/RadeonOpenCompute/rocm_bandwidth_test")
if (DEFINED ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
   set(CPACK_DEBIAN_PACKAGE_RELEASE $ENV{CPACK_DEBIAN_PACKAGE_RELEASE})
else()
   set(CPACK_DEBIAN_PACKAGE_RELEASE "local")
endif()

# RPM package specific variables
set(CPACK_RPM_PACKAGE_REQUIRES "hsa-rocr")
if(DEFINED CPACK_PACKAGING_INSTALL_PREFIX)
  set(CPACK_RPM_EXCLUDE_FROM_AUTO_FILELIST_ADDITION "${CPACK_PACKAGING_INSTALL_PREFIX} ${CPACK_PACKAGING_INSTALL_PREFIX}/${CMAKE_INSTALL_BINDIR}")
endif()
if(DEFINED ENV{CPACK_RPM_PACKAGE_RELEASE})
  set(CPACK_RPM_PACKAGE_RELEASE $ENV{CPACK_RPM_PACKAGE_RELEASE})
else()
  set(CPACK_RPM_PACKAGE_RELEASE "local")
endif()

# Set rpm distro
if(CPACK_RPM_PACKAGE_RELEASE)
  set(CPACK_RPM_PACKAGE_RELEASE_DIST ON)
endif()

# Prepare final version for the CAPACK use
set(CPACK_PACKAGE_VERSION "${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}.${ROCM_VERSION_FOR_PACKAGE}")

# Set the names now using CPACK utility
set(CPACK_DEBIAN_FILE_NAME "DEB-DEFAULT")
set(CPACK_RPM_FILE_NAME "RPM-DEFAULT")

include(CPack)
