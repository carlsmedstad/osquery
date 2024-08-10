# Copyright (c) 2014-present, The osquery authors
#
# This source code is licensed as defined by the LICENSE file found in the
# root directory of this source tree.
#
# SPDX-License-Identifier: (Apache-2.0 OR GPL-2.0-only)

cmake_minimum_required(VERSION 3.21.4 FATAL_ERROR)

# toolchain.cmake needs to be included before project() because the former sets the compiler path for the custom toolchain,
# if the user specify it and the latter does compiler detection.
# utilities.cmake is a dependency of toolchain.cmake.
include(cmake/utilities.cmake)
include(cmake/toolchain.cmake)

project(osquery)

if(OSQUERY_BUILD_TESTS)
  enable_testing()
endif()

include(cmake/globals.cmake)
include(cmake/options.cmake)
include(cmake/ccache.cmake)
include(cmake/flags.cmake)
include("${OSQUERY_INSTALL_DIRECTIVES}")

if(OSQUERY_TOOLCHAIN_SYSROOT AND NOT DEFINED PLATFORM_LINUX)
  message(FATAL_ERROR "The custom toolchain can only be used with Linux, undefine OSQUERY_TOOLCHAIN_SYSROOT and specify a compiler to use")
endif()

function(main)
  findClangFormat()
  findPythonExecutablePath()
  generateSpecialTargets()

  if(OSQUERY_ENABLE_FORMAT_ONLY)
    return()
  endif()

  message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")
  message(STATUS "Shared libraries: ${BUILD_SHARED_LIBS}")

  if(DEFINED PLATFORM_MACOS)
    if((NOT "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" AND NOT "${CMAKE_C_COMPILER_ID}" STREQUAL "AppleClang") OR
      (NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" AND NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "AppleClang"))
      message(STATUS "Warning: the selected C or C++ compiler is not clang/clang++. Compilation may fail")
    endif()
  elseif(NOT DEFINED PLATFORM_WINDOWS)
    if(NOT "${CMAKE_C_COMPILER_ID}" STREQUAL "Clang" OR
      NOT "${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
      message(STATUS "Warning: the selected C or C++ compiler is not clang/clang++. Compilation may fail")
    endif()
  endif()

  add_subdirectory("libraries")
  importLibraries()

  # Enable clang-tidy only after having created the third party targets,
  # so it won't run with those and slow down the build.
  if(OSQUERY_ENABLE_CLANG_TIDY)
    find_package(clang-tidy)

    if(TARGET clang-tidy::clang-tidy)
      set(CMAKE_CXX_CLANG_TIDY
        "${CLANG-TIDY_EXECUTABLE};${OSQUERY_CLANG_TIDY_CHECKS}"
      )
    else()
      message(WARNING "clang-tidy: Disabled because it was not found")
    endif()
  endif()

  add_subdirectory("osquery")
  add_subdirectory("plugins")
  add_subdirectory("tools")
  add_subdirectory("specs")
  add_subdirectory("external")
  add_subdirectory("tests")

  if(DEFINED PLATFORM_WINDOWS)
    enableOsqueryWEL()
  endif()

  generateInstallDirectives()
endfunction()

function(add_system_pkgconfig_dependency osquery_name system_name)
  find_package(PkgConfig REQUIRED)
  pkg_check_modules(${osquery_name} REQUIRED ${system_name})
  add_library("thirdparty_${osquery_name}" INTERFACE)
  add_real_target_dependencies(thirdparty_libraries "thirdparty_${osquery_name}")
  target_link_libraries("thirdparty_${osquery_name}" INTERFACE ${${osquery_name}_LIBRARIES})
  target_include_directories("thirdparty_${osquery_name}" INTERFACE ${${osquery_name}_INCLUDE_DIRS})
endfunction()

function(add_system_cmake_dependency)
  cmake_parse_arguments(ARG "" "NAME;OSQUERY_NAME" "COMPONENTS;LINK_LIBRARIES" ${ARGN})
  if(ARG_OSQUERY_NAME)
    string(TOLOWER "${ARG_OSQUERY_NAME}" osquery_name)
  else()
    string(TOLOWER "thirdparty_${ARG_NAME}" osquery_name)
  endif()
  if(ARG_COMPONENTS)
    find_package(${ARG_NAME} REQUIRED COMPONENTS ${ARG_COMPONENTS})
  else()
    find_package(${ARG_NAME} REQUIRED)
  endif()
  add_library(${osquery_name} INTERFACE)
  add_real_target_dependencies(thirdparty_libraries ${osquery_name})
  if(ARG_LINK_LIBRARIES)
    target_link_libraries(${osquery_name} INTERFACE ${ARG_LINK_LIBRARIES})
  endif()
endfunction()

function(importLibraries)
  set(system_library_descriptor_list
    # Leave lz4 at the top (see LZ4 CMakeLists.txt)
    "Linux,Darwin,Windows:lz4:liblz4"
    "Linux,Darwin:augeas:augeas"
    "Linux,Darwin,Windows:bzip2:bzip2"
    "Linux,Darwin,Windows:gflags:gflags"
    "Linux,Darwin,Windows:libarchive:libarchive"
    "Linux:libaudit:audit"
    "Linux:libcryptsetup:libcryptsetup"
    "Linux:libdevmapper:devmapper"
    "Linux:libgcrypt:libgcrypt"
    "Linux:libgpg-error:gpg-error"
    "Linux:libiptables:libiptc"
    "Linux,Darwin:libmagic:libmagic"
    "Linux,Darwin,Windows:librdkafka:rdkafka"
    "Linux:librpm:rpm"
    "Linux:libudev:libudev"
    "Linux,Darwin,Windows:lzma:liblzma"
    "Linux:popt:popt"
    "Linux,Darwin,Windows:rapidjson:RapidJSON"
    "Linux,Darwin,Windows:rocksdb:rocksdb"
    "Linux,Darwin,Windows:sleuthkit:tsk"
    "Linux,Darwin,Windows:sqlite:sqlite3"
    "Linux,Darwin,Windows:thrift:thrift"
    "Linux,Darwin,Windows:yara:yara"
    "Linux,Darwin,Windows:zlib:zlib"
    "Linux,Darwin,Windows:zstd:libzstd"
    "Linux:expat:expat"
    "Linux:dbus:dbus-1"
    "Linux:libcap:libcap"
  )
  set(library_descriptor_list)
  if(NOT OSQUERY_USE_SYSTEM_LIBRARIES)
    list(APPEND library_descriptor_list
      ${system_library_descriptor_list}
      "Linux,Darwin,Windows:openssl"
      "Linux,Darwin,Windows:boost"
      "Linux,Darwin,Windows:glog"
    )
  endif()
  list(APPEND library_descriptor_list
    "Linux,Darwin,Windows:linenoise-ng"
    "Linux:util-linux"
  )

  if(OSQUERY_BUILD_BPF)
    list(APPEND library_descriptor_list
      "Linux:ebpfpub"
    )
  endif()

  if(OSQUERY_BUILD_AWS)
    if(NOT OSQUERY_USE_SYSTEM_LIBRARIES)
      list(APPEND library_descriptor_list
        "Linux,Darwin,Windows:aws-sdk-cpp"
      )
    endif()
  endif()

  if(OSQUERY_BUILD_DPKG)
    list(APPEND library_descriptor_list
      "Linux:libdpkg"
    )
  endif()
  
  if(OSQUERY_BUILD_ETW)
    list(APPEND library_descriptor_list
      "Windows:krabsetw"
    )
  endif()  

  # Create a target that builds all the third party libraries for convenience
  add_custom_target(thirdparty_libraries)

  if(OSQUERY_USE_SYSTEM_LIBRARIES)
    add_system_cmake_dependency(
      NAME OpenSSL
      LINK_LIBRARIES OpenSSL::SSL OpenSSL::Crypto
    )
    add_system_cmake_dependency(
      NAME Boost
      COMPONENTS filesystem system chrono thread context
      LINK_LIBRARIES Boost::chrono Boost::context Boost::filesystem Boost::system Boost::thread
    )
    add_system_cmake_dependency(
      NAME glog
      LINK_LIBRARIES glog::glog
    )
    add_system_cmake_dependency(
      NAME GTest
      OSQUERY_NAME thirdparty_googletest
      LINK_LIBRARIES GTest::gmock GTest::gmock_main GTest::gtest GTest::gtest_main
    )
    add_system_cmake_dependency(
      NAME GTest
      OSQUERY_NAME thirdparty_googletest_headers
    )

    add_system_cmake_dependency(
      NAME aws-cpp-sdk-core
      LINK_LIBRARIES aws-cpp-sdk-core
    )
    add_system_cmake_dependency(
      NAME aws-cpp-sdk-sts
      LINK_LIBRARIES aws-cpp-sdk-sts thirdparty_aws-cpp-sdk-core
    )
    add_system_cmake_dependency(
      NAME aws-cpp-sdk-kinesis
      LINK_LIBRARIES aws-cpp-sdk-kinesis thirdparty_aws-cpp-sdk-core
    )
    add_system_cmake_dependency(
      NAME aws-cpp-sdk-firehose
      LINK_LIBRARIES aws-cpp-sdk-firehose thirdparty_aws-cpp-sdk-core
    )
    add_system_cmake_dependency(
      NAME aws-cpp-sdk-ec2
      LINK_LIBRARIES aws-cpp-sdk-ec2 thirdparty_aws-cpp-sdk-core
    )

    foreach(library_descriptor ${system_library_descriptor_list})
      string(REPLACE ":" ";" library_descriptor "${library_descriptor}")
      list(GET library_descriptor 0 platform_list)
      list(GET library_descriptor 1 library)
      list(GET library_descriptor 2 pkgconfig_library)
      add_system_pkgconfig_dependency(${library} ${pkgconfig_library})
    endforeach()

    set_property(GLOBAL PROPERTY AUGEAS_LENSES_FOLDER_PATH "/usr/share/augeas/lenses/dist")
  endif()

  foreach(library_descriptor ${library_descriptor_list})
    # Expand the library descriptor
    string(REPLACE ":" ";" library_descriptor "${library_descriptor}")

    list(GET library_descriptor 0 platform_list)
    list(GET library_descriptor 1 library)

    string(REPLACE "," ";" platform_list "${platform_list}")

    list(FIND platform_list "${CMAKE_SYSTEM_NAME}" platform_index)

    if(platform_index EQUAL -1)
      continue()
    endif()

    find_package("${library}" REQUIRED)

    # Skip libraries which already use our internal target name
    if(TARGET "thirdparty_${library}")
      add_real_target_dependencies(thirdparty_libraries "thirdparty_${library}")
      continue()

    # For generic libraries that import the library name, let's create
    # an alias
    elseif(TARGET "${library}")
      add_library("thirdparty_${library}" ALIAS "${library}")
      add_real_target_dependencies(thirdparty_libraries "thirdparty_${library}")

    # Legacy libraries will just export variables; build a new INTERFACE
    # target with them
    elseif(DEFINED "${library}_LIBRARIES")
      if(NOT DEFINED "${library}_INCLUDE_DIRS")
        message(FATAL_ERROR "Variable ${library}_INCLUDE_DIRS was not found!")
      endif()

      add_library("thirdparty_${library}" INTERFACE)
      add_real_target_dependencies(thirdparty_libraries "thirdparty_${library}")

      target_link_libraries("thirdparty_${library}" INTERFACE
        ${library}_LIBRARIES
      )

      target_include_directories("thirdparty_${library}" INTERFACE
        ${library}_INCLUDE_DIRS
      )

      if(DEFINED "${library}_DEFINITIONS")
        target_compile_definitions("thirdparty_${library}" INTERFACE
          ${library}_DEFINITIONS
        )
      endif()

    else()
      # In case we were trying to import ebpfpub, check whether the build option
      # has been turned OFF automatically because the installed LLVM libraries
      # were broken/not compatible
      if(NOT OSQUERY_BUILD_BPF AND "${library}" STREQUAL "ebpfpub")
        message(WARNING "ebpfpub could not correctly import the LLVM libraries. BPF support has been disabled")
      else()
        message(FATAL_ERROR "The '${library}' library was found but it couldn't be imported correctly")
      endif()
    endif()
  endforeach()
endfunction()

main()
