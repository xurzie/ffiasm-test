cmake_minimum_required(VERSION 3.20)
project(ffiasm_tests LANGUAGES C CXX ASM_NASM)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)

# GMP
find_package(PkgConfig REQUIRED)
pkg_check_modules(GMP REQUIRED IMPORTED_TARGET gmp)

# paths
set(FFIASM_GEN_DIR ${CMAKE_SOURCE_DIR})  # fr.* and fq.* in root
set(FFIASM_VENDOR ${CMAKE_SOURCE_DIR}/third_party/ffiasm_upstream)

include_directories(${FFIASM_GEN_DIR})
include_directories(${FFIASM_VENDOR}/c)
include_directories(${FFIASM_VENDOR}/test)

# building generated fields (ASM + C++)
# Linux: nasm -> ELF64; if macOS — need -fmacho64 and prefix
enable_language(ASM_NASM)
if(APPLE)
  set(CMAKE_ASM_NASM_OBJECT_FORMAT macho64)
  add_compile_options($<$<COMPILE_LANGUAGE:ASM_NASM>:--prefix _>)
else()
  set(CMAKE_ASM_NASM_OBJECT_FORMAT elf64)
endif()

add_library(ffiasm_fields STATIC
  ${FFIASM_GEN_DIR}/fr.cpp
  ${FFIASM_GEN_DIR}/fr.asm
  ${FFIASM_GEN_DIR}/fq.cpp
  ${FFIASM_GEN_DIR}/fq.asm
)

target_link_libraries(ffiasm_fields PRIVATE PkgConfig::GMP)

# googletest
include(FetchContent)
FetchContent_Declare(
  googletest
  URL https://github.com/google/googletest/archive/refs/tags/v1.14.0.zip
)
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

# build tests
set(FFIASM_UPSTREAM_C_SOURCES
  ${FFIASM_VENDOR}/c/alt_bn128.cpp
  ${FFIASM_VENDOR}/c/splitparstr.cpp   # splitParStr(...)
  ${FFIASM_VENDOR}/c/naf.cpp           # buildNaf(...)
  ${FFIASM_VENDOR}/c/misc.cpp          # log2(unsigned)
)

add_library(ffiasm_upstream STATIC ${FFIASM_UPSTREAM_C_SOURCES})
target_include_directories(ffiasm_upstream PUBLIC
  ${FFIASM_VENDOR}/c
  ${FFIASM_GEN_DIR}                # fr.hpp / fq.hpp от buildzqfield
  ${FFIASM_VENDOR}/test/tester
)
target_link_libraries(ffiasm_upstream PRIVATE PkgConfig::GMP)

# === tests ===
file(GLOB_RECURSE FFIASM_UPSTREAM_TEST_SOURCES
  ${FFIASM_VENDOR}/test/*.cpp
)
list(REMOVE_ITEM FFIASM_UPSTREAM_TEST_SOURCES
  ${FFIASM_VENDOR}/test/tester/tester.cpp
)

add_executable(alt_bn128_tests
  ${FFIASM_VENDOR}/c/alt_bn128_test.cpp
  ${FFIASM_UPSTREAM_TEST_SOURCES}
)

target_link_libraries(alt_bn128_tests
  PRIVATE
    ffiasm_fields
    ffiasm_upstream
    gtest
    PkgConfig::GMP
)

# --- CTest + GTest discovery ---
include(CTest)
enable_testing()

include(GoogleTest)
gtest_discover_tests(alt_bn128_tests
  DISCOVERY_TIMEOUT 60
  WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

if (MSVC)
  target_compile_options(alt_bn128_tests PRIVATE /O2)
else()
  target_compile_options(alt_bn128_tests PRIVATE -O2 -march=native)
endif()

include(GoogleTest)
gtest_discover_tests(alt_bn128_tests)
