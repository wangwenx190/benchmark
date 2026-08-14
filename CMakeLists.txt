cmake_minimum_required(VERSION 4.0)

project(benchmark VERSION 1.9.5 LANGUAGES CXX)

if(WIN32)
  enable_language(RC)
endif()

include(GNUInstallDirs)

option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
option(BENCHMARK_ENABLE_EXCEPTIONS "Enable exceptions" ON)
option(BENCHMARK_ENABLE_LTO "Enable link-time optimization" OFF)
option(BENCHMARK_USE_LIBCXX "Use libc++ with Clang" OFF)
option(BENCHMARK_ENABLE_WERROR "Treat warnings as errors" ON)
option(BENCHMARK_FORCE_WERROR "Force warnings as errors" OFF)
option(BENCHMARK_ENABLE_INSTALL "Enable installation" OFF)
option(BENCHMARK_ENABLE_LIBPFM "Enable libpfm performance counters" OFF)

if(MSVC OR CMAKE_CXX_SIMULATE_ID STREQUAL "MSVC")
  set(BENCHMARK_BUILD_32_BITS OFF CACHE BOOL "Build a 32-bit library" FORCE)
else()
  option(BENCHMARK_BUILD_32_BITS "Build a 32-bit library" OFF)
endif()

if(BENCHMARK_FORCE_WERROR)
  set(BENCHMARK_ENABLE_WERROR ON)
endif()

file(GLOB benchmark_sources CONFIGURE_DEPENDS
  "${CMAKE_CURRENT_SOURCE_DIR}/src/*.cc"
  "${CMAKE_CURRENT_SOURCE_DIR}/src/*.h"
  "${CMAKE_CURRENT_SOURCE_DIR}/include/benchmark/*.h")
list(REMOVE_ITEM benchmark_sources
  "${CMAKE_CURRENT_SOURCE_DIR}/src/benchmark_main.cc")

add_library(benchmark ${benchmark_sources})
add_library(benchmark::benchmark ALIAS benchmark)
set_target_properties(benchmark PROPERTIES
  OUTPUT_NAME benchmark
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR})
target_compile_features(benchmark PUBLIC cxx_std_17)
target_include_directories(benchmark
  PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
  PRIVATE
    "${CMAKE_CURRENT_SOURCE_DIR}/src")
target_compile_definitions(benchmark
  PRIVATE
    BENCHMARK_VERSION="v${PROJECT_VERSION}"
    HAVE_STEADY_CLOCK)

if(CMAKE_SYSTEM_NAME MATCHES "^(Linux|FreeBSD)$")
  target_compile_definitions(benchmark PRIVATE
    BENCHMARK_HAS_PTHREAD_AFFINITY)
endif()

add_library(benchmark_main "src/benchmark_main.cc")
add_library(benchmark::benchmark_main ALIAS benchmark_main)
set_target_properties(benchmark_main PROPERTIES
  OUTPUT_NAME benchmark_main
  VERSION ${PROJECT_VERSION}
  SOVERSION ${PROJECT_VERSION_MAJOR}
  DEFINE_SYMBOL benchmark_EXPORTS)
target_compile_features(benchmark_main PUBLIC cxx_std_17)
target_link_libraries(benchmark_main PUBLIC benchmark::benchmark)

if(NOT BUILD_SHARED_LIBS)
  target_compile_definitions(benchmark PUBLIC BENCHMARK_STATIC_DEFINE)
endif()

set(THREADS_PREFER_PTHREAD_FLAG ON)
find_package(Threads REQUIRED)
target_link_libraries(benchmark PRIVATE Threads::Threads)

if(WIN32)
  target_link_libraries(benchmark PRIVATE shlwapi)
elseif(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
  target_link_libraries(benchmark PRIVATE kstat)
elseif(UNIX AND NOT APPLE AND NOT CMAKE_SYSTEM_NAME STREQUAL "QNX")
  target_link_libraries(benchmark PRIVATE rt)
endif()

if(BENCHMARK_ENABLE_LIBPFM)
  find_path(PFM_INCLUDE_DIR perfmon/pfmlib.h REQUIRED)
  find_library(PFM_LIBRARY NAMES pfm REQUIRED)
  target_include_directories(benchmark PRIVATE "${PFM_INCLUDE_DIR}")
  target_link_libraries(benchmark PRIVATE "${PFM_LIBRARY}")
  target_compile_definitions(benchmark PRIVATE HAVE_LIBPFM)
endif()

foreach(target IN ITEMS benchmark benchmark_main)
  set_target_properties(${target} PROPERTIES
    CXX_EXTENSIONS OFF
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN ON)

  if(BENCHMARK_ENABLE_LTO)
    set_property(TARGET ${target} PROPERTY INTERPROCEDURAL_OPTIMIZATION ON)
  endif()

  if(MSVC)
    target_compile_options(${target} PRIVATE /W4 /utf-8 /permissive-)
    if(CMAKE_CXX_COMPILER_ID STREQUAL "MSVC")
      target_compile_options(${target} PRIVATE /MP)
    endif()
    target_compile_definitions(${target} PRIVATE _CRT_SECURE_NO_WARNINGS)
    if(BENCHMARK_ENABLE_WERROR)
      target_compile_options(${target} PRIVATE /WX)
    endif()
    if(NOT BENCHMARK_ENABLE_EXCEPTIONS)
      target_compile_options(${target} PRIVATE /EHs- /EHa-)
      target_compile_definitions(${target} PRIVATE _HAS_EXCEPTIONS=0)
    endif()
  else()
    target_compile_options(${target} PRIVATE
      -Wall -Wextra -Wshadow -Wfloat-equal -Wold-style-cast -Wconversion
      -Wformat=2 -Wpedantic -Wno-deprecated-declarations)
    target_compile_definitions(${target} PRIVATE
      _FILE_OFFSET_BITS=64 _LARGEFILE64_SOURCE _LARGEFILE_SOURCE)
    if(BENCHMARK_ENABLE_WERROR)
      target_compile_options(${target} PRIVATE -Werror -pedantic-errors)
    endif()
    if(NOT BENCHMARK_ENABLE_EXCEPTIONS)
      target_compile_options(${target} PRIVATE -fno-exceptions)
    endif()
    if(BENCHMARK_BUILD_32_BITS)
      target_compile_options(${target} PRIVATE -m32)
      target_link_options(${target} PRIVATE -m32)
    endif()
    if(CYGWIN)
      target_compile_definitions(${target} PRIVATE _GNU_SOURCE=1)
    elseif(CMAKE_SYSTEM_NAME STREQUAL "QNX")
      target_compile_definitions(${target} PRIVATE _QNX_SOURCE)
    endif()
  endif()

  if(BENCHMARK_USE_LIBCXX)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
      target_compile_options(${target} PRIVATE -stdlib=libc++)
      target_link_options(${target} PRIVATE -stdlib=libc++)
    else()
      message(FATAL_ERROR "BENCHMARK_USE_LIBCXX requires Clang")
    endif()
  endif()
endforeach()

if(WIN32 AND BUILD_SHARED_LIBS)
  target_sources(benchmark PRIVATE benchmark.rc)
  target_sources(benchmark_main PRIVATE benchmark_main.rc)
endif()

if(BENCHMARK_ENABLE_INSTALL)
  install(TARGETS benchmark benchmark_main
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    INCLUDES DESTINATION ${CMAKE_INSTALL_INCLUDEDIR})
  install(DIRECTORY include/benchmark
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    FILES_MATCHING PATTERN "*.h")
endif()
