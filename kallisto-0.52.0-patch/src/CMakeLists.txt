file(GLOB sources *.cpp)
file(GLOB headers *.h *.hpp)

list(REMOVE_ITEM sources main.cpp)

if (USE_BAM)
include_directories(${HTSLIB_INCLUDE_DIRS})
endif(USE_BAM)

if(NOT MAX_KMER_SIZE)
set(MAX_KMER_SIZE "32")
endif()

add_compile_options(-Wno-subobject-linkage) # Suppress bifrost warning

add_library(kallisto_core ${sources} ${headers})
target_include_directories(kallisto_core PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})
if(USE_BAM)
    # MSYS2/UCRT64: htslib may come from the active MinGW prefix instead of the
    # vendored ext/htslib tree, so use the library path selected by top-level CMake.
    target_include_directories(kallisto_core PUBLIC ${HTSLIB_INCLUDE_DIRS})
endif()
if(USE_HDF5)
    # MSYS2/UCRT64: HDF5 headers are resolved explicitly from the active MinGW
    # prefix to avoid mixing MSYS and UCRT64 headers.
    target_include_directories(kallisto_core PUBLIC ${HDF5_INCLUDE_DIRS})
endif()
# MSYS2/UCRT64: kallisto_core includes bifrost headers generated/configured by
# ExternalProject, so build bifrost before compiling the core library.
add_dependencies(kallisto_core bifrost)

add_executable(kallisto main.cpp ${sources} ${headers})
if(USE_BAM)
    target_include_directories(kallisto PRIVATE ${HTSLIB_INCLUDE_DIRS})
endif()
if(USE_HDF5)
    target_include_directories(kallisto PRIVATE ${HDF5_INCLUDE_DIRS})
endif()
# MSYS2/UCRT64: bifrost is an ExternalProject, so make the executable wait for
# its static library before the link step.
add_dependencies(kallisto bifrost)

find_package( Threads REQUIRED )
ExternalProject_Get_Property(bifrost binary_dir)
if (USE_BAM)
target_link_libraries(kallisto kallisto_core pthread ${HTSLIB_LIBRARIES} ${binary_dir}/src/libbifrost.a  -DMAX_KMER_SIZE=${MAX_KMER_SIZE})
else()
target_link_libraries(kallisto kallisto_core pthread ${binary_dir}/src/libbifrost.a -DMAX_KMER_SIZE=${MAX_KMER_SIZE})
endif(USE_BAM)

if(LINK MATCHES static)
    set(BUILD_SHARED_LIBS OFF)
    set(HDF5_USE_STATIC_LIBRARIES 1)

    if (UNIX AND NOT APPLE)
        #set(CMAKE_EXE_LINKER_FLAGS  "-static -static-libgcc -static-libstdc++")
        set(CMAKE_EXE_LINKER_FLAGS  "-static -static-libstdc++")
        SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
        set(CMAKE_EXE_LINKER_FLAGS  "-static -static-libgcc -static-libstdc++")
    endif(UNIX AND NOT APPLE)

    SET_TARGET_PROPERTIES(kallisto  PROPERTIES LINK_SEARCH_END_STATIC 1)
endif(LINK MATCHES static)


if(USE_HDF5)
    #find_package( HDF5 COMPONENTS CXX REQUIRED )
endif(USE_HDF5)

if (NOT ZLIBNG)
if(MINGW)
    # MSYS2/UCRT64: prefer zlib from the same MinGW prefix as the compiler so
    # MSYS headers from C:/msys64/usr/include are not mixed into UCRT64 builds.
    get_filename_component(KALLISTO_MINGW_BIN_DIR "${CMAKE_CXX_COMPILER}" DIRECTORY)
    get_filename_component(KALLISTO_MINGW_PREFIX "${KALLISTO_MINGW_BIN_DIR}" DIRECTORY)
    if(EXISTS "${KALLISTO_MINGW_PREFIX}/include/zlib.h")
        set(ZLIB_INCLUDE_DIR "${KALLISTO_MINGW_PREFIX}/include" CACHE PATH "zlib include directory" FORCE)
        if(EXISTS "${KALLISTO_MINGW_PREFIX}/lib/libz.dll.a")
            set(ZLIB_LIBRARY "${KALLISTO_MINGW_PREFIX}/lib/libz.dll.a" CACHE FILEPATH "zlib import library" FORCE)
            set(ZLIB_LIBRARY_RELEASE "${KALLISTO_MINGW_PREFIX}/lib/libz.dll.a" CACHE FILEPATH "zlib release import library" FORCE)
        elseif(EXISTS "${KALLISTO_MINGW_PREFIX}/lib/libz.a")
            set(ZLIB_LIBRARY "${KALLISTO_MINGW_PREFIX}/lib/libz.a" CACHE FILEPATH "zlib static library" FORCE)
            set(ZLIB_LIBRARY_RELEASE "${KALLISTO_MINGW_PREFIX}/lib/libz.a" CACHE FILEPATH "zlib release static library" FORCE)
        endif()
    endif()
endif()
find_package( ZLIB REQUIRED )

if ( ZLIB_FOUND )
    if (DEFINED ZLIB_INCLUDE_DIRS)
        include_directories( ${ZLIB_INCLUDE_DIRS} )
    elseif (DEFINED ZLIB_INCLUDE_DIR)
        include_directories( ${ZLIB_INCLUDE_DIR} )
    else()
        message(FATAL_ERROR "zlib found but no include directories are set.")
    endif()
    target_link_libraries(kallisto kallisto_core ${ZLIB_LIBRARIES})
else()
    message(FATAL_ERROR "zlib not found. Required for to output files" )
endif( ZLIB_FOUND )
endif(NOT ZLIBNG)

if (ZLIBNG)
    add_definitions(-DKALLISTO_USE_ZLIB_NG)
    include_directories(../ext/zlib-ng/zlib-ng)
    include_directories(../ext/zlib-ng)
    if(WIN32)
    target_link_libraries(kallisto ${CMAKE_CURRENT_SOURCE_DIR}/../ext/zlib-ng/zlib-ng/libz.lib)
    else()
    target_link_libraries(kallisto ${CMAKE_CURRENT_SOURCE_DIR}/../ext/zlib-ng/zlib-ng/libz.a)
    endif(WIN32)
endif(ZLIBNG)

if(USE_HDF5)
    #if(HDF5_FOUND)
        include_directories( ${HDF5_INCLUDE_DIRS} )
        #target_link_libraries( kallisto_core ${HDF5_LIBRARIES} )
        target_link_libraries( kallisto ${HDF5_LIBRARIES} )
    #else()
    #    message(FATAL_ERROR "HDF5 not found. Required to output files")
    #endif()
endif(USE_HDF5)

if(LINK MATCHES static)
    if (UNIX AND NOT APPLE)
        target_link_libraries(kallisto librt.a)
    endif()
else()
    if (UNIX AND NOT APPLE)
        target_link_libraries(kallisto rt)
    endif()
endif(LINK MATCHES static)


install(TARGETS kallisto DESTINATION "${CMAKE_INSTALL_BINDIR}")
