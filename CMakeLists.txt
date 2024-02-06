# Copyright (c) Microsoft Corporation. All rights reserved.
# Licensed under the MIT license.

# Parameters:
#
# BOOST_ROOT:
#   Specify root of the Boost library if Boost cannot be auto-detected. On Windows, a fallback to a
#   downloaded nuget version will be used if Boost cannot be found.
#
# DISKANN_RELEASE_UNUSED_TCMALLOC_MEMORY_AT_CHECKPOINTS:
#   This is a work-in-progress feature, not completed yet. The core DiskANN library will be split into
#   build-related and search-related functionality. In build-related functionality, when using tcmalloc,
#   it's possible to release memory that's free but reserved by tcmalloc. Setting this to true enables
#   such behavior.
#   Contact for this feature: gopalrs.


# Some variables like MSVC are defined only after project(), so put that first.
cmake_minimum_required(VERSION 3.15)
project(diskann)

set(CMAKE_STANDARD 17)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

if(NOT MSVC)
	set(CMAKE_CXX_COMPILER g++)
endif()

set(CMAKE_MODULE_PATH "${PROJECT_SOURCE_DIR}/cmake;${CMAKE_MODULE_PATH}")


include_directories(${PROJECT_SOURCE_DIR}/include)

find_package(OpenMP REQUIRED)
set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")

find_package(Boost REQUIRED)
include_directories(${Boost_INCLUDE_DIR})

find_package(BLAS REQUIRED)
link_libraries(${BLAS_LIBRARIES})

find_package(LAPACK REQUIRED)
link_libraries(${LAPACK_LIBRARIES})

set(DISKANN_ASYNC_LIB aio)

if (CMAKE_SYSTEM_PROCESSOR MATCHES "amd64|x86_64")
    if (CMAKE_LIBRARY_ARCHITECTURE MATCHES "i386")
        message (FATAL_ERROR "32bit platforms are not supported")
    endif ()
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mavx2 -mfma -msse2")
elseif (CMAKE_SYSTEM_PROCESSOR MATCHES "^(aarch64.*|AARCH64.*|arm64.*|ARM64.*)")
    # TODO: ARM64 specific flags
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -ftree-vectorize -fno-builtin-malloc -fno-builtin-calloc -fno-builtin-realloc -fno-builtin-free -fopenmp -fopenmp-simd -funroll-loops -Wfatal-errors")
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -g -DDEBUG")
if (NOT PYBIND)
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -DNDEBUG -Ofast")
    if (NOT PORTABLE)
        set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -march=native -mtune=native")
    endif()
else()
    # -Ofast is not supported in a python extension module
    set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -DNDEBUG -fPIC")
endif()

add_subdirectory(src)

if (UNIT_TEST)
    enable_testing()
    add_subdirectory(tests)
endif()

include(clang-format.cmake)