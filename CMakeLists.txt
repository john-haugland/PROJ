################################################################################
#
# This file is part of CMake configuration for PROJ library (inspired from SOCI
# CMake,  Copyright (C) 2009-2010 Mateusz Loskot <mateusz@loskot.net> )
#
# Copyright (C) 2011 Nicolas David <nicolas.david@ign.fr>
# Distributed under the MIT license
#
################################################################################
# General settings
################################################################################
cmake_minimum_required(VERSION 3.9 FATAL_ERROR)

project(PROJ
  DESCRIPTION "PROJ coordinate transformation software library"
  LANGUAGES C CXX
)

# Only interpret if() arguments as variables or keywords when unquoted
cmake_policy(SET CMP0054 NEW)

# Set C++ version
# Make CMAKE_CXX_STANDARD available as cache option overridable by user
set(CMAKE_CXX_STANDARD 11
  CACHE STRING "C++ standard version to use (default is 11)")
message(STATUS "Requiring C++${CMAKE_CXX_STANDARD}")
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
message(STATUS "Requiring C++${CMAKE_CXX_STANDARD} - done")

# Set C99 version
# Make CMAKE_C_STANDARD available as cache option overridable by user
set(CMAKE_C_STANDARD 99
  CACHE STRING "C standard version to use (default is 99)")
message(STATUS "Requiring C${CMAKE_C_STANDARD}")
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_C_EXTENSIONS OFF)
message(STATUS "Requiring C${CMAKE_C_STANDARD} - done")

set(CMAKE_SKIP_BUILD_RPATH FALSE)
set(CMAKE_BUILD_WITH_INSTALL_RPATH FALSE)
set(CMAKE_INSTALL_RPATH_USE_LINK_PATH TRUE)
if(APPLE)
    set(MACOSX_RPATH ON)
endif()

# Set global -fvisibility=hidden
set(CMAKE_C_VISIBILITY_PRESET hidden)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

# Set warnings as variables, then store as cache options
set(PROJ_common_WARN_FLAGS  # common only to GNU/Clang C/C++
  -Wall
  -Wextra
  -Wswitch
  -Wshadow
  -Wunused-parameter
  -Wmissing-declarations
  -Wformat
  -Wformat-security
)
if("${CMAKE_C_COMPILER_ID}" STREQUAL "GNU")
  set(PROJ_C_WARN_FLAGS ${PROJ_common_WARN_FLAGS}
    -Wmissing-prototypes
  )
  set(PROJ_CXX_WARN_FLAGS ${PROJ_common_WARN_FLAGS})
elseif("${CMAKE_C_COMPILER_ID}" MATCHES "Clang")
  set(PROJ_C_WARN_FLAGS ${PROJ_common_WARN_FLAGS}
    -Wmissing-prototypes
    -Wfloat-conversion
    -Wc11-extensions
  )
  set(PROJ_CXX_WARN_FLAGS ${PROJ_common_WARN_FLAGS}
    -Wfloat-conversion
  )
elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "MSVC")
  add_definitions(/D_CRT_SECURE_NO_WARNINGS) # Eliminate deprecation warnings
  set(PROJ_C_WARN_FLAGS
    /W4
    /wd4706  # Suppress warning about assignment within conditional expression
    /wd4996  # Suppress warning about sprintf, etc., being unsafe
  )
  set(PROJ_CXX_WARN_FLAGS /EHsc ${PROJ_C_WARN_FLAGS})
elseif("${CMAKE_C_COMPILER_ID}" STREQUAL "Intel")
  if(MSVC)
    set(PROJ_C_WARN_FLAGS /Wall)
    set(PROJ_CXX_WARN_FLAGS /Wall)
  else()
    set(PROJ_C_WARN_FLAGS -Wall)
    set(PROJ_CXX_WARN_FLAGS -Wall)
  endif()
endif()

set(PROJ_C_WARN_FLAGS "${PROJ_C_WARN_FLAGS}"
  CACHE STRING "C flags used to compile PROJ targets")
set(PROJ_CXX_WARN_FLAGS "${PROJ_CXX_WARN_FLAGS}"
  CACHE STRING "C++ flags used to compile PROJ targets")

################################################################################
# PROJ CMake modules
################################################################################
# Path to additional CMake modules
set(CMAKE_MODULE_PATH ${PROJ_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

include(ProjUtilities)

message(STATUS "Configuring PROJ:")

################################################################################
#PROJ version information
################################################################################
include(ProjVersion)
proj_version(MAJOR 8 MINOR 2 PATCH 0)
set(PROJ_API_VERSION "22")
set(PROJ_BUILD_VERSION "23.0.1")

################################################################################
# Build features and variants
################################################################################
include(ProjConfig)
include(ProjMac)
include(policies)

################################################################################
# Check for nlohmann_json
################################################################################

set(NLOHMANN_JSON_ORIGIN "auto" CACHE STRING
"nlohmann/json origin. The default auto will try to use external \
nlohmann/json if possible")
set_property(CACHE NLOHMANN_JSON_ORIGIN PROPERTY STRINGS auto internal external)

# Probably not the strictest minimum, but known to work with it
set(MIN_NLOHMANN_JSON_VERSION 3.7.0)

if(NLOHMANN_JSON_ORIGIN STREQUAL "external")
  find_package(nlohmann_json REQUIRED)
  set(NLOHMANN_JSON "external")
elseif(NLOHMANN_JSON_ORIGIN STREQUAL "internal")
  set(NLOHMANN_JSON "internal")
else()
  find_package(nlohmann_json QUIET)
  if(nlohmann_json_FOUND)
    set(NLOHMANN_JSON "external")
  else()
    set(NLOHMANN_JSON "internal")
  endif()
endif()

if(NLOHMANN_JSON STREQUAL "external")
  # Check minimum version
  if(nlohmann_json_VERSION VERSION_LESS MIN_NLOHMANN_JSON_VERSION)
    message(STATUS "external nlohmann/json version ${nlohmann_json_VERSION} "
      "is older than minimum requirement ${MIN_NLOHMANN_JSON_VERSION}")
    set(NLOHMANN_JSON "internal")
  else()
    message(STATUS "found nlohmann/json version ${nlohmann_json_VERSION}")
  endif()
endif()

message(STATUS "nlohmann/json: ${NLOHMANN_JSON}")

################################################################################
# Check for sqlite3
################################################################################

find_program(EXE_SQLITE3 sqlite3)
if(NOT EXE_SQLITE3)
  message(SEND_ERROR "sqlite3 binary not found!")
endif()

find_package(Sqlite3 REQUIRED)
if(NOT SQLITE3_FOUND)
  message(SEND_ERROR "sqlite3 dependency not found!")
endif()

# Would build and run with older versions, but with horrible performance
# See https://github.com/OSGeo/PROJ/issues/1718
if("${SQLITE3_VERSION}" VERSION_LESS "3.11")
  message(SEND_ERROR "sqlite3 >= 3.11 required!")
endif()

################################################################################
# Check for libtiff
################################################################################

option(ENABLE_TIFF "Enable TIFF support to read some grids" ON)
mark_as_advanced(ENABLE_TIFF)
set(TIFF_ENABLED FALSE)
if(ENABLE_TIFF)
  find_package(TIFF REQUIRED)
  if(TIFF_FOUND)
    set(TIFF_ENABLED TRUE)
  else()
    message(SEND_ERROR
      "libtiff dependency not found! Use ENABLE_TIFF=OFF to force it off")
  endif()
else()
  message(WARNING
    "TIFF support is not enabled and will result in the inability to read "
    "some grids")
endif()

################################################################################
# Check for curl
################################################################################

option(ENABLE_CURL "Enable Curl support" ON)
set(CURL_ENABLED FALSE)
if(ENABLE_CURL)
  find_package(CURL REQUIRED)
  if(CURL_FOUND)
    set(CURL_ENABLED TRUE)
  else()
    message(SEND_ERROR "curl dependency not found!")
  endif()
endif()

################################################################################

option(PROJ_LIB_ENV_VAR_TRIED_LAST "Whether the PROJ_LIB environment variable should be tried after the hardcoded location" OFF)
if(PROJ_LIB_ENV_VAR_TRIED_LAST)
    add_definitions(-DPROJ_LIB_ENV_VAR_TRIED_LAST)
endif()

################################################################################
# threading configuration
################################################################################
set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
find_package(Threads)

include(CheckCSourceCompiles)
if(MSVC)
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_C_FLAGS} /WX /W4")
else()
  set(CMAKE_REQUIRED_LIBRARIES m)
  set(CMAKE_REQUIRED_FLAGS "${CMAKE_C_FLAGS} -Werror -Wall")
endif()

if(Threads_FOUND AND CMAKE_USE_PTHREADS_INIT)
  set(CMAKE_REQUIRED_LIBRARIES
    "${CMAKE_REQUIRED_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT}")
  check_c_source_compiles("
#include <pthread.h>

int main(int argc, char* argv[]) {
  (void)PTHREAD_MUTEX_RECURSIVE;
  (void)argv;
  return argc;
}
  " HAVE_PTHREAD_MUTEX_RECURSIVE_DEFN)
  if(HAVE_PTHREAD_MUTEX_RECURSIVE_DEFN)
    add_definitions(-DHAVE_PTHREAD_MUTEX_RECURSIVE=1)
  endif()
endif()

# Set a default build type for single-configuration cmake generators if
# no build type is set.
if(NOT CMAKE_CONFIGURATION_TYPES AND NOT CMAKE_BUILD_TYPE)
  set(CMAKE_BUILD_TYPE Release)
endif()

if(MSVC OR CMAKE_CONFIGURATION_TYPES)
  # For multi-config systems and for Visual Studio, the debug version of
  # the library has _d appended.
  set(CMAKE_DEBUG_POSTFIX _d)
endif()

# Put the libraries and binaries that get built into directories at the
# top of the build tree rather than in hard-to-find leaf
# directories. This simplifies manual testing and the use of the build
# tree rather than installed PROJ libraries.
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${PROJ_BINARY_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${PROJ_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${PROJ_BINARY_DIR}/bin)
link_directories(${CMAKE_LIBRARY_OUTPUT_DIRECTORY})

################################################################################
# Installation
################################################################################
include(ProjInstallPath)
set(BINDIR "${DEFAULT_BINDIR}"
  CACHE PATH "The directory to install binaries into.")
set(LIBDIR "${DEFAULT_LIBDIR}"
  CACHE PATH "The directory to install libraries into.")
set(DATADIR "${DEFAULT_DATADIR}"
  CACHE PATH "The directory to install data files into.")
set(DOCDIR "${DEFAULT_DOCDIR}"
  CACHE PATH "The directory to install doc files into.")
set(INCLUDEDIR "${DEFAULT_INCLUDEDIR}"
  CACHE PATH "The directory to install includes into.")
set(CMAKECONFIGDIR "${DEFAULT_CMAKEDIR}"
  CACHE PATH "Parent of the directory to install cmake config files into.")

################################################################################
# Tests
################################################################################
include(CTest)

if(BUILD_TESTING)
  include(ProjTest)
else()
  message(STATUS "Testing disabled")
endif()

################################################################################
# Build configured components
################################################################################
include_directories(${PROJ_SOURCE_DIR}/src)

add_subdirectory(data)
add_subdirectory(include)
add_subdirectory(src)
add_subdirectory(man)
add_subdirectory(cmake)
if(BUILD_TESTING)
  add_subdirectory(test)
endif()

set(docfiles COPYING NEWS AUTHORS)
install(FILES ${docfiles}
        DESTINATION "${DOCDIR}")

################################################################################
# pkg-config support
################################################################################
if(UNIX OR MINGW)
  configure_proj_pc()

  install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/proj.pc
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/pkgconfig)
endif()

################################################################################
# "make dist" workalike
################################################################################

set(CPACK_SOURCE_GENERATOR "TGZ;ZIP")
set(CPACK_SOURCE_PACKAGE_FILE_NAME "proj-${PROJ_VERSION}")
set(CPACK_PACKAGE_VENDOR "OSGeo")
set(CPACK_PACKAGE_VERSION_MAJOR ${PROJ_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROJ_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROJ_VERSION_PATCH})
set(CPACK_VERBATIM_VARIABLES TRUE)
set(CPACK_SOURCE_IGNORE_FILES
  /\\..*  # any file/directory starting with .
  /.*\\.yml
  /.*\\.gz
  /.*\\.zip
  /.*build.*/
  \\.deps
  /autogen\\.sh
  /autom4te\\.cache
  /CODE_OF_CONDUCT.md
  /CONTRIBUTING.md
  /Dockerfile
  /docs/
  /Doxyfile
  /examples/
  /HOWTO-RELEASE
  /m4/lt*
  /m4/libtool*
  /media/
  /schemas/
  /scripts/
  /test/fuzzers/
  /test/gigs/.*gie\\.failing
  /test/postinstall/
  /travis/
  ${PROJECT_BINARY_DIR}
)

include(CPack)

# Simplify README.md to README
add_custom_target(README
  COMMAND ${CMAKE_COMMAND}
    -D PROJ_SOURCE_DIR=${PROJ_SOURCE_DIR}
    -P ${PROJ_SOURCE_DIR}/cmake/ProjReadme.cmake
)

get_property(_is_multi_config_generator GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT _is_multi_config_generator)
  add_custom_target(dist
    COMMAND ${CMAKE_MAKE_PROGRAM} package_source
    DEPENDS README
  )
  message(STATUS "PROJ: Configured 'dist' target")
endif()
