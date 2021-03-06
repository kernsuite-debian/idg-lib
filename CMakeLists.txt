cmake_minimum_required(VERSION 2.8.0)

# set policies
cmake_policy(SET CMP0003 NEW) # 2.6.0
cmake_policy(SET CMP0012 NEW) # 2.8.0

# the project
project(ImageDomainGridder CXX)

# set cmake module path
list(APPEND CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake/modules")


# options
option (BUILD_STATIC_LIBS "Build static libraries" OFF)
option (BUILD_LIB_CPU "Build CPU libraries" ON)
option (BUILD_LIB_CUDA "Build CUDA libraries" OFF)
option (BUILD_LIB_OPENCL "Build OpenCL libraries" OFF)
option (BUILD_WITH_PYTHON "Build Python bindings" OFF)
option (BUILD_WITH_MKL "Build with MKL (if available)" OFF)
option (BUILD_WITH_POWERSENSOR " Build with PowerSensor" OFF)
option (REPORT_VERBOSE "Report detailed performance metrics" OFF)
option (REPORT_TOTAL "Report total performance metrics" OFF)
option (REPORT_OPS "Report performance in ops (fma, sincos) instead of flops (add, mul)" OFF)
option (COMPILE_VERBOSE "Report compiler flags used during runtime compilation" OFF)

# build shared or static libraries (default: shared)
if (BUILD_STATIC_LIBS)
set (BUILD_SHARED_LIBS FALSE)
else()
set (BUILD_SHARED_LIBS TRUE)
endif()

# Installation
if (CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX ${CMAKE_BINARY_DIR} CACHE PATH
    "IDG install prefix" FORCE)
endif()


# Compiler settings:
if (NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
  set(CMAKE_BUILD_TYPE Release CACHE STRING "Default build type." FORCE)
endif()
# set debug print info
set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} -DDEBUG")

if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang")
  set(CLANG_CXX_COMPILER TRUE)
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  set(GNU_CXX_COMPILER TRUE)
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel")
  set(INTEL_CXX_COMPILER TRUE)
  set (CMAKE_SHARED_LINKER_FLAGS "-static-intel")
elseif ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
  set(MSVC_CXX_COMPILER TRUE)
endif()

# make sure C++11 is used (in newer CMake use CXX_STANDARD)
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -std=c++0x")
else()
    message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

# make sure openmp is used if available
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Clang" AND ${CMAKE_CXX_COMPILER_VERSION} VERSION_LESS "3.8.0")
    set (OpenMP_CXX_FLAGS "-fopenmp")
    set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
    set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
else()
    find_package(OpenMP REQUIRED)
    if (OPENMP_FOUND)
        if (INTEL_CXX_COMPILER)
        set (OpenMP_CXX_FLAGS "-qopenmp")
        endif()
        set (CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${OpenMP_C_FLAGS}")
        set (CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${OpenMP_CXX_FLAGS}")
        set (CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${OpenMP_EXE_LINKER_FLAGS}")
    else()
        message(STATUS "OpenMP support not detected")
    endif()
endif()

# detect architecture
execute_process(COMMAND uname -m COMMAND tr -d '\n' OUTPUT_VARIABLE ARCHITECTURE)
if (${ARCHITECTURE} STREQUAL "ppc64le")
  set(POWER_ARCHITECTURE TRUE)
endif()

# resolve dependencies
if (BUILD_WITH_POWERSENSOR)
find_package(PowerSensor QUIET)
endif ()
if (${POWERSENSOR_FOUND})
    set(HAVE_POWERSENSOR TRUE)
    include_directories(${POWERSENSOR_INCLUDE_DIR})
else ()
    unset(POWERSENSOR_LIBRARY CACHE)
    unset(POWERSENSOR_INCLUDE_DIR CACHE)
endif()

# FFT library
if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "Intel" OR ${BUILD_WITH_MKL})
# assume MKL is present when using Intel compiler
find_package(MKL QUIET)
set(HAVE_MKL TRUE)
set(FFTW3_LIBRARIES ${MKL_LIBRARIES})
set(FFTW3_INCLUDE_DIR "${MKL_INCLUDE_DIRS}/fftw")
else()
find_package(FFTW3 COMPONENTS single threads)
endif()

include_directories (${CMAKE_CURRENT_BINARY_DIR}/src)

# add main include folder
include_directories (${CMAKE_CURRENT_SOURCE_DIR}/src)

# add subfolders, after specifying the targets to build
add_subdirectory(src)

# write file to initialize environment
configure_file (
  "${PROJECT_SOURCE_DIR}/cmake/config/init-environment.sh.in"
  "${PROJECT_BINARY_DIR}/init-environment.sh"
  )

install(
    FILES
    ${PROJECT_BINARY_DIR}/init-environment.sh
    DESTINATION
    ${CMAKE_INSTALL_PREFIX}
)

# write config file, included in all files of the project
configure_file (
    "${PROJECT_SOURCE_DIR}/cmake/config/idg-config.h.in"
    "${PROJECT_BINARY_DIR}/src/idg-config.h"
)

install(
    FILES
    ${PROJECT_BINARY_DIR}/src/idg-config.h
    DESTINATION
    ${CMAKE_INSTALL_PREFIX}/include
)

# Create idg-targets.cmake
set(TARGETS_TO_EXPORT idg)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-common)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-fft)
if (BUILD_LIB_CPU)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-cpu)
endif()
if (BUILD_LIB_CUDA)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-cuda)
endif()
if (BUILD_LIB_CPU AND BUILD_LIB_CUDA)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-hybrid-cuda)
endif()
if (BUILD_LIB_OPENCL)
set(TARGETS_TO_EXPORT ${TARGETS_TO_EXPORT} idg-opencl)
endif()
export(
    TARGETS ${TARGETS_TO_EXPORT}
    FILE ${PROJECT_BINARY_DIR}/CMakeFiles/idg-targets.cmake
)

install(
    EXPORT idg-targets
    DESTINATION lib/cmake
)

# Create idg-config.cmake
configure_file(
    ${PROJECT_SOURCE_DIR}/cmake/config/idg-config.cmake.in
    ${PROJECT_BINARY_DIR}/CMakeFiles/idg-config.cmake @ONLY
)

install(
    FILES ${PROJECT_BINARY_DIR}/CMakeFiles/idg-config.cmake
    DESTINATION ${CMAKE_INSTALL_PREFIX}
)
