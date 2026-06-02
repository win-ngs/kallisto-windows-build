# MSYS2/UCRT64 ships CMake 4.x, which rejects old policy compatibility;
# 3.10 also avoids the deprecation warning emitted by current CMake.
cmake_minimum_required(VERSION 3.10)

project(kallisto)

include(GNUInstallDirs)

if(NOT MAX_KMER_SIZE)
    set(MAX_KMER_SIZE "32")
endif()

if(MINGW)
    # MSYS2/UCRT64 release builds should not depend on the build machine's CPU
    # features; let users opt back in explicitly for local optimized builds.
    if(NOT DEFINED ENABLE_AVX2)
        set(ENABLE_AVX2 "OFF" CACHE STRING "Enable AVX2 instructions")
    endif()
    if(NOT DEFINED COMPILATION_ARCH)
        set(COMPILATION_ARCH "OFF" CACHE STRING "Native CPU architecture tuning")
    endif()
endif()

set(DO_ENABLE_AVX2 "")
set(DO_ENABLE_COMPILATION_ARCH "")

if(ENABLE_AVX2 MATCHES "OFF")
    add_compile_definitions("ENABLE_AVX2=OFF")
    set(DO_ENABLE_AVX2 "-DENABLE_AVX2=OFF")
endif(ENABLE_AVX2 MATCHES "OFF")

if(COMPILATION_ARCH MATCHES "OFF")
    add_compile_definitions("COMPILATION_ARCH=OFF")
    set(DO_ENABLE_COMPILATION_ARCH "-DCOMPILATION_ARCH=OFF")
endif(COMPILATION_ARCH MATCHES "OFF")

add_compile_definitions("MAX_KMER_SIZE=${MAX_KMER_SIZE}")


option(USE_HDF5 "Compile with HDF5 support" OFF) #OFF by default
option(USE_BAM "Compile with HTSLIB support" OFF) # OFF by default

if(USE_HDF5)
    cmake_policy(SET CMP0074 NEW)
    add_compile_definitions("USE_HDF5=ON")
endif(USE_HDF5)

if(NOT USE_BAM)
    add_compile_definitions("NO_HTSLIB=ON")
endif()

set(EXT_PROJECTS_DIR ${PROJECT_SOURCE_DIR}/ext)
set(CMAKE_CXX_FLAGS_PROFILE "-g")

if(MINGW)
    # MSYS2/UCRT64: derive the active MinGW prefix from the selected compiler so
    # optional libraries are taken from UCRT64, not from the MSYS POSIX runtime.
    get_filename_component(KALLISTO_MINGW_BIN_DIR "${CMAKE_CXX_COMPILER}" DIRECTORY)
    get_filename_component(KALLISTO_MINGW_PREFIX "${KALLISTO_MINGW_BIN_DIR}" DIRECTORY)
endif()

# Set Release type for builds where CMAKE_BUILD_TYPE is unset
# This is usually a good default as this implictly enables
#
#   CXXFLAGS = -O3 -DNDEBUG
#
if(NOT CMAKE_BUILD_TYPE)
    set(CMAKE_BUILD_TYPE "Release")
endif()

if(CMAKE_BUILD_TYPE MATCHES "Debug")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O0 -g")
endif()

if(${CMAKE_VERSION} VERSION_LESS 3.1)
    # CMake versions before 3.1 do not understand CMAKE_CXX_STANDARD
    # remove this block once CMake >=3.1 has fixated in the ecosystem
    add_compile_options(-std=c++11)
else()
    include(CheckCXXCompilerFlag)
    check_cxx_compiler_flag(-std=c++17 COMPILER_SUPPORTS_CXX17)
    if(COMPILER_SUPPORTS_CXX17)
        set(CMAKE_CXX_STANDARD 17)
    else()
        set(CMAKE_CXX_STANDARD 11)
    endif()

    set(CMAKE_CXX_STANDARD_REQUIRED ON)
    set(CMAKE_CXX_EXTENSIONS OFF)
endif()

#add_compile_options(-Wall -Wno-unused-function)
add_compile_options(-Wno-deprecated-declarations -Wno-subobject-linkage) # Suppress bifrost warning
set(PROJECT_BIFROST_CMAKE_CXX_FLAGS "-Wno-subobject-linkage -Wno-return-type") # Suppress bifrost warning

if(LINK MATCHES static)
    message("static build")
ELSE(LINK MATCHES shared)
    message("shared build")
ENDIF(LINK MATCHES static)


include(ExternalProject)
if (USE_BAM)
    if(MINGW)
        # MSYS2/UCRT64: use the packaged native htslib import library. The
        # vendored htslib ExternalProject does not build libhts.a on Windows.
        find_path(HTSLIB_INCLUDE_DIR
            NAMES htslib/sam.h
            PATHS "${KALLISTO_MINGW_PREFIX}/include"
            NO_DEFAULT_PATH)
        find_library(HTSLIB_LIBRARY
            NAMES hts
            PATHS "${KALLISTO_MINGW_PREFIX}/lib"
            NO_DEFAULT_PATH)
        if(NOT HTSLIB_INCLUDE_DIR OR NOT HTSLIB_LIBRARY)
            message(FATAL_ERROR "HTSlib was requested but was not found in ${KALLISTO_MINGW_PREFIX}. Install mingw-w64-ucrt-x86_64-htslib.")
        endif()
        set(HTSLIB_INCLUDE_DIRS "${HTSLIB_INCLUDE_DIR}")
        set(HTSLIB_LIBRARIES "${HTSLIB_LIBRARY}")
    else()
        ExternalProject_Add(htslib
            PREFIX ${PROJECT_SOURCE_DIR}/ext/htslib
            SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/htslib
            BUILD_IN_SOURCE 1
            CONFIGURE_COMMAND "" #autoreconf -i && autoheader &&  autoconf && ${PROJECT_SOURCE_DIR}/ext/htslib/configure
                #--prefix=${PREFIX} --disable-bz2 --disable-lzma --disable-libcurl
            BUILD_COMMAND "" #make lib-static
            INSTALL_COMMAND ""
        )
        set(HTSLIB_INCLUDE_DIRS "${PROJECT_SOURCE_DIR}/ext/htslib")
        set(HTSLIB_LIBRARIES "${PROJECT_SOURCE_DIR}/ext/htslib/libhts.a")
    endif()
endif(USE_BAM)

if(MINGW)
    # MSYS2/UCRT64: pass zlib from the active MinGW compiler prefix into
    # ExternalProject builds so their CMake runs do not pick MSYS zlib.
    if(EXISTS "${KALLISTO_MINGW_PREFIX}/include/zlib.h")
        if(EXISTS "${KALLISTO_MINGW_PREFIX}/lib/libz.dll.a")
            set(KALLISTO_MINGW_ZLIB_LIBRARY "${KALLISTO_MINGW_PREFIX}/lib/libz.dll.a")
        elseif(EXISTS "${KALLISTO_MINGW_PREFIX}/lib/libz.a")
            set(KALLISTO_MINGW_ZLIB_LIBRARY "${KALLISTO_MINGW_PREFIX}/lib/libz.a")
        endif()

        if(KALLISTO_MINGW_ZLIB_LIBRARY)
            set(KALLISTO_EXTERNAL_ZLIB_ARGS
                -DZLIB_INCLUDE_DIR=${KALLISTO_MINGW_PREFIX}/include
                -DZLIB_LIBRARY=${KALLISTO_MINGW_ZLIB_LIBRARY}
                -DZLIB_LIBRARY_RELEASE=${KALLISTO_MINGW_ZLIB_LIBRARY})
        endif()
    endif()
endif()

set(BIFROST_BINARY_DIR ${PROJECT_BINARY_DIR}/bifrost-build)

ExternalProject_Add(bifrost
    # MSYS2/UCRT64: keep ExternalProject stamps under the current build tree;
    # sharing stamps from the source tree makes separate build directories think
    # Bifrost is already built even when their libbifrost.a does not exist.
    PREFIX ${PROJECT_BINARY_DIR}/bifrost-prefix
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/bifrost
    BINARY_DIR ${BIFROST_BINARY_DIR}
    # MSYS2/UCRT64: keep bifrost on the same generator, make program, and
    # compiler prefix as kallisto; otherwise CMake may choose Ninja/MSYS tools
    # while the parent build is using UCRT64 Makefiles.
    CONFIGURE_COMMAND
        ${CMAKE_COMMAND}
            -S <SOURCE_DIR>
            -B <BINARY_DIR>
            -G ${CMAKE_GENERATOR}
            -DCMAKE_MAKE_PROGRAM=${CMAKE_MAKE_PROGRAM}
            -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
            -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
            -DCMAKE_POLICY_VERSION_MINIMUM=3.10
            -DMAX_KMER_SIZE=${MAX_KMER_SIZE}
            -DCMAKE_INSTALL_PREFIX=<INSTALL_DIR>
            -DCMAKE_CXX_FLAGS=${PROJECT_BIFROST_CMAKE_CXX_FLAGS}
            ${KALLISTO_EXTERNAL_ZLIB_ARGS}
            ${DO_ENABLE_AVX2}
            ${DO_ENABLE_COMPILATION_ARCH}
    BUILD_COMMAND ${CMAKE_COMMAND} --build <BINARY_DIR>
    INSTALL_COMMAND ""
)

if (ZLIBNG)
    message("zlib-ng enabled.")
    ExternalProject_Add(zlib-ng
    PREFIX ${PROJECT_SOURCE_DIR}/ext/zlib-ng
    SOURCE_DIR ${PROJECT_SOURCE_DIR}/ext/zlib-ng
    BUILD_IN_SOURCE 1
    CONFIGURE_COMMAND mkdir -p zlib-ng && cd zlib-ng && cmake .. -DZLIB_COMPAT=ON -DZLIB_ENABLE_TESTS=OFF -DCMAKE_INSTALL_PREFIX=${PREFIX}
    BUILD_COMMAND cd zlib-ng && make
    INSTALL_COMMAND ""
    )
endif(ZLIBNG)

if (USE_BAM)
    include_directories(${HTSLIB_INCLUDE_DIRS})
endif(USE_BAM)

if(USE_HDF5)
    if(MINGW)
        # MSYS2/UCRT64: find HDF5 in the active UCRT64 prefix explicitly. The
        # default CMake search path can otherwise prefer MSYS headers/libs.
        find_path(HDF5_INCLUDE_DIR
            NAMES hdf5.h
            PATHS "${KALLISTO_MINGW_PREFIX}/include"
            NO_DEFAULT_PATH)
        find_library(HDF5_LIBRARY
            NAMES hdf5
            PATHS "${KALLISTO_MINGW_PREFIX}/lib"
            NO_DEFAULT_PATH)
        if(NOT HDF5_INCLUDE_DIR OR NOT HDF5_LIBRARY)
            message(FATAL_ERROR "HDF5 was requested but was not found in ${KALLISTO_MINGW_PREFIX}. Install mingw-w64-ucrt-x86_64-hdf5.")
        endif()
        set(HDF5_INCLUDE_DIRS "${HDF5_INCLUDE_DIR}")
        set(HDF5_LIBRARIES "${HDF5_LIBRARY}")
    else()
        find_package(HDF5 REQUIRED)
    endif()
endif(USE_HDF5)

ExternalProject_Get_Property(bifrost binary_dir)
include_directories(${binary_dir}/src)
ExternalProject_Get_Property(bifrost source_dir)
# MSYS2/UCRT64: Bifrost headers live in the ExternalProject source tree, while
# generated/configured files are in the binary tree; include both explicitly.
include_directories(${source_dir}/src)




add_subdirectory(src)
include_directories(${EXT_PROJECTS_DIR})

option(BUILD_TESTING "Build unit tests." OFF)
include(CTest)

if (BUILD_TESTING)
    add_subdirectory(${EXT_PROJECTS_DIR}/catch)

    # Includes Catch in the project:
    include_directories(${CATCH_INCLUDE_DIR} ${COMMON_INCLUDES})

    add_subdirectory(unit_tests)
endif(BUILD_TESTING)

option(BUILD_FUNCTESTING "Build functional tests." OFF)

if (BUILD_FUNCTESTING)
    add_subdirectory(func_tests)
    message("Functional testing enabled.")
    add_custom_target(test
    COMMAND /bin/bash ./func_tests/runtests.sh
    DEPENDS ./src/kallisto
    )
endif(BUILD_FUNCTESTING)

# enable_testing()
# add_test(MainTest test/tests)
