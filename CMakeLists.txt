# Copyright 2021 Google LLC
#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#      https://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.

cmake_policy(SET CMP0012 NEW)
cmake_policy(SET CMP0048 NEW)
project(ruy CXX)
cmake_minimum_required(VERSION 3.13)  # Copied from IREE
set(CMAKE_CXX_STANDARD 14)

include(GNUInstallDirs)

if (PROJECT_NAME STREQUAL CMAKE_PROJECT_NAME)
  set(RUY_IS_TOPLEVEL TRUE)
  set(RUY_MINIMAL_BUILD_DEFAULT_VALUE OFF)
else()
  set(RUY_IS_TOPLEVEL FALSE)
  set(RUY_MINIMAL_BUILD_DEFAULT_VALUE ON)
endif()

option(RUY_MINIMAL_BUILD "Disable ruy's tests, examples, etc. Build only ruy public libraries." ${RUY_MINIMAL_BUILD_DEFAULT_VALUE})
if (NOT RUY_MINIMAL_BUILD)
  enable_testing()
endif()

option(RUY_PROFILER "Enable ruy's built-in profiler (harms performance)" OFF)

option(RUY_ENABLE_INSTALL "Enable install rule" ${RUY_IS_TOPLEVEL})

if(${CMAKE_HOST_SYSTEM_PROCESSOR} MATCHES "arm*")
  if(CMAKE_SYSTEM_PROCESSOR MATCHES "^(arm.*|ARM.*)")  # 32bit
    add_compile_options("-mfpu=neon")  # we'll assume it
    message(STATUS "ARM 32bit, adding -mfpu=neon")
  elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64.*|AARCH64.*)")  # NEON is implied on aarch64
    if(NOT "${CMAKE_SIZEOF_VOID_P}" EQUAL "8")
      message(FATAL_ERROR "invalid pointer size")
    endif()
    message(STATUS "AARCH64")
  else()
    message(FATAL_ERROR "Unknown ARM processor '${CMAKE_SYSTEM_PROCESSOR}'")
  endif()
  set(NEON_FOUND true CACHE BOOL "NEON available on host")
  set(ARM_FOUND true CACHE BOOL "arm")
endif()

include(cmake/ruy_add_all_subdirs.cmake)
include(cmake/ruy_cc_library.cmake)
include(cmake/ruy_cc_binary.cmake)
include(cmake/ruy_cc_test.cmake)

find_package(PkgConfig REQUIRED)
pkg_check_modules(CPUINFO REQUIRED IMPORTED_TARGET libcpuinfo)

# custom imported target for cpuinfo because we are not using CMake
# https://git.maemo.org/leste/bugtracker/issues/819
add_library(cpuinfo::cpuinfo INTERFACE IMPORTED)
target_link_libraries(cpuinfo::cpuinfo INTERFACE PkgConfig::CPUINFO)
target_include_directories(cpuinfo::cpuinfo INTERFACE ${CPUINFO_INCLUDE_DIRS})

# googletest is only needed for tests. Projects embedding ruy as a subdirectory
# and not needing to build ruy tests may proceed without a local checkout of
# third_party/googletest.
if (NOT RUY_MINIMAL_BUILD
    AND NOT TARGET gtest
    AND EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/third_party/googletest/CMakeLists.txt")
  add_subdirectory("third_party/googletest" EXCLUDE_FROM_ALL)
endif()

add_subdirectory("ruy")

if (NOT RUY_MINIMAL_BUILD)
  add_subdirectory("example")
endif()

if (RUY_ENABLE_INSTALL)
  install(EXPORT ${PROJECT_NAME}Targets
    NAMESPACE ${PROJECT_NAME}::
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
  )

  include(CMakePackageConfigHelpers)

  configure_package_config_file(
    "cmake/${PROJECT_NAME}Config.cmake.in"
    "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    INSTALL_DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
  )

  install(FILES "${PROJECT_BINARY_DIR}/${PROJECT_NAME}Config.cmake"
    DESTINATION "${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}"
  )
endif()
