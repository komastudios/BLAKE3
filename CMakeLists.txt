cmake_minimum_required(VERSION 3.21)

project(blake3 LANGUAGES C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_STANDARD_REQUIRED True)

set(CMAKE_POSITION_INDEPENDENT_CODE ON)

option(BUILD_SHARED "Build shared library" ON)
option(USE_MSVC_STATIC_RUNTIME "Use MSVC static runtime" OFF)

if(CMAKE_C_COMPILER_ID MATCHES "GNU|Clang|AppleClang")
    add_compile_options("-ffile-prefix-map=${CMAKE_CURRENT_SOURCE_DIR}=.")
    add_compile_options("-fdebug-prefix-map=${CMAKE_CURRENT_SOURCE_DIR}=.")
endif()

if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    add_compile_options("/Brepro")
    add_link_options("/Brepro")
    add_compile_options("/pathmap:${CMAKE_CURRENT_SOURCE_DIR}=.")
endif()

if(${CMAKE_SYSTEM_NAME} MATCHES "Android")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")

    add_compile_options(-fsigned-char)
    add_link_options("-Wl,-z,max-page-size=16384")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "iOS")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")
elseif(${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
    set(BUILD_SHARED OFF)

    set(CMAKE_C_FLAGS_RELWITHDEBINFO "-O2 -g -DNDEBUG")
    set(CMAKE_C_FLAGS_RELEASE "-O2 -g1 -DNDEBUG")
endif()

if(CMAKE_C_COMPILER_ID MATCHES "MSVC")
    if(USE_MSVC_STATIC_RUNTIME)
        message(STATUS "Using MSVC static runtime")
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    else()
        message(STATUS "Using MSVC dynamic runtime")
        set(CMAKE_MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
    endif()
endif()

# Core sources (always compiled)
set(LIB_SOURCE_FILES
    c/blake3.c
    c/blake3_dispatch.c
    c/blake3_portable.c)

# Architecture detection
set(BLAKE3_AMD64_NAMES amd64 AMD64 x86_64)
set(BLAKE3_X86_NAMES i686 x86 X86)
set(BLAKE3_ARMv8_NAMES aarch64 AArch64 arm64 ARM64 armv8 armv8a)

# Determine BLAKE3_ARCH
# Emscripten reports x86 but doesn't support x86 SIMD — use portable only
if(${CMAKE_SYSTEM_NAME} MATCHES "Emscripten")
    # BLAKE3_ARCH left unset — portable fallback
# On Apple, check CMAKE_OSX_ARCHITECTURES first (cross-compile target vs host)
elseif(APPLE AND DEFINED CMAKE_OSX_ARCHITECTURES AND NOT CMAKE_OSX_ARCHITECTURES STREQUAL "")
    string(TOLOWER "${CMAKE_OSX_ARCHITECTURES}" _osx_arch_lower)
    if(_osx_arch_lower MATCHES "x86_64")
        set(BLAKE3_ARCH "amd64")
    elseif(_osx_arch_lower MATCHES "arm64")
        set(BLAKE3_ARCH "arm64")
    endif()
elseif(MSVC AND DEFINED CMAKE_C_COMPILER_ARCHITECTURE_ID)
    if(CMAKE_C_COMPILER_ARCHITECTURE_ID MATCHES "[Xx]64")
        set(BLAKE3_ARCH "amd64")
    elseif(CMAKE_C_COMPILER_ARCHITECTURE_ID MATCHES "[Xx]86")
        set(BLAKE3_ARCH "x86")
    elseif(CMAKE_C_COMPILER_ARCHITECTURE_ID MATCHES "[Aa][Rr][Mm]64")
        set(BLAKE3_ARCH "arm64")
    endif()
else()
    if(CMAKE_SYSTEM_PROCESSOR IN_LIST BLAKE3_AMD64_NAMES)
        set(BLAKE3_ARCH "amd64")
    elseif(CMAKE_SYSTEM_PROCESSOR IN_LIST BLAKE3_X86_NAMES)
        set(BLAKE3_ARCH "x86")
    elseif(CMAKE_SYSTEM_PROCESSOR IN_LIST BLAKE3_ARMv8_NAMES)
        if(CMAKE_SIZEOF_VOID_P EQUAL 8)
            set(BLAKE3_ARCH "arm64")
        else()
            set(BLAKE3_ARCH "armv7")
        endif()
    elseif(ANDROID_ABI STREQUAL "armeabi-v7a")
        set(BLAKE3_ARCH "armv7")
    endif()
endif()

message(STATUS "BLAKE3_ARCH: ${BLAKE3_ARCH}")

# SIMD source selection based on BLAKE3_ARCH
if(BLAKE3_ARCH STREQUAL "amd64")
    if(MSVC)
        enable_language(ASM_MASM)
        list(APPEND LIB_SOURCE_FILES
            c/blake3_avx2_x86-64_windows_msvc.asm
            c/blake3_avx512_x86-64_windows_msvc.asm
            c/blake3_sse2_x86-64_windows_msvc.asm
            c/blake3_sse41_x86-64_windows_msvc.asm)
    elseif(WIN32 OR CYGWIN)
        enable_language(ASM)
        list(APPEND LIB_SOURCE_FILES
            c/blake3_avx2_x86-64_windows_gnu.S
            c/blake3_avx512_x86-64_windows_gnu.S
            c/blake3_sse2_x86-64_windows_gnu.S
            c/blake3_sse41_x86-64_windows_gnu.S)
    elseif(UNIX)
        enable_language(ASM)
        list(APPEND LIB_SOURCE_FILES
            c/blake3_avx2_x86-64_unix.S
            c/blake3_avx512_x86-64_unix.S
            c/blake3_sse2_x86-64_unix.S
            c/blake3_sse41_x86-64_unix.S)
    endif()

elseif(BLAKE3_ARCH STREQUAL "x86")
    list(APPEND LIB_SOURCE_FILES
        c/blake3_avx2.c
        c/blake3_avx512.c
        c/blake3_sse2.c
        c/blake3_sse41.c)
    if(MSVC)
        set_source_files_properties(c/blake3_sse2.c PROPERTIES COMPILE_FLAGS "/arch:SSE2")
        set_source_files_properties(c/blake3_sse41.c PROPERTIES COMPILE_FLAGS "/arch:AVX")
        set_source_files_properties(c/blake3_avx2.c PROPERTIES COMPILE_FLAGS "/arch:AVX2")
        set_source_files_properties(c/blake3_avx512.c PROPERTIES COMPILE_FLAGS "/arch:AVX512")
    else()
        set_source_files_properties(c/blake3_sse2.c PROPERTIES COMPILE_FLAGS "-msse2")
        set_source_files_properties(c/blake3_sse41.c PROPERTIES COMPILE_FLAGS "-msse4.1")
        set_source_files_properties(c/blake3_avx2.c PROPERTIES COMPILE_FLAGS "-mavx2")
        set_source_files_properties(c/blake3_avx512.c PROPERTIES COMPILE_FLAGS "-mavx512f -mavx512vl")
    endif()

elseif(BLAKE3_ARCH STREQUAL "arm64")
    list(APPEND LIB_SOURCE_FILES c/blake3_neon.c)

elseif(BLAKE3_ARCH STREQUAL "armv7")
    list(APPEND LIB_SOURCE_FILES c/blake3_neon.c)
    set_source_files_properties(c/blake3_neon.c PROPERTIES COMPILE_FLAGS "-mfpu=neon")
endif()

# Library target function (mirrors lz4 pattern)
function(add_blake3_library TARGET_NAME LIB_TYPE)
    add_library(${TARGET_NAME} ${LIB_TYPE} ${LIB_SOURCE_FILES})
    target_include_directories(${TARGET_NAME} PUBLIC ${CMAKE_CURRENT_SOURCE_DIR}/c)
    set_property(TARGET ${TARGET_NAME} PROPERTY C_VISIBILITY_PRESET hidden)

    if(BLAKE3_ARCH STREQUAL "armv7")
        target_compile_definitions(${TARGET_NAME} PRIVATE BLAKE3_USE_NEON=1)
    elseif(NOT DEFINED BLAKE3_ARCH OR BLAKE3_ARCH STREQUAL "")
        target_compile_definitions(${TARGET_NAME} PRIVATE
            BLAKE3_USE_NEON=0
            BLAKE3_NO_SSE2
            BLAKE3_NO_SSE41
            BLAKE3_NO_AVX2
            BLAKE3_NO_AVX512)
    endif()

    if(LIB_TYPE STREQUAL "SHARED")
        if(WIN32)
            target_compile_definitions(${TARGET_NAME} PRIVATE BLAKE3_DLL_EXPORTS)
            target_compile_definitions(${TARGET_NAME} PUBLIC BLAKE3_DLL)
        endif()
    endif()

    install(TARGETS ${TARGET_NAME}
        ARCHIVE DESTINATION lib
        LIBRARY DESTINATION lib
        RUNTIME DESTINATION bin)
endfunction()

add_blake3_library(blake3-static STATIC)

install(FILES c/blake3.h
    DESTINATION include)

if(BUILD_SHARED)
    add_blake3_library(blake3 SHARED)
endif()
