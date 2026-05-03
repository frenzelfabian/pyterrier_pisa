cmake_minimum_required(VERSION 3.15)

project(${SKBUILD_PROJECT_NAME} LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

# Architecture-specific compiler flags. SSE/AVX intrinsics only exist on x86.
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(amd64)|(AMD64)")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mtune=generic -march=x86-64 -msse4.2 -mavx -mavx2 -mfma -mbmi -mrdseed -mpopcnt")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "(aarch64)|(arm64)|(ARM64)")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mtune=generic")
endif()

# Speed up the build by removing CLI tools, tests, and benchmarks
set(PISA_BUILD_TOOLS OFF CACHE BOOL "" FORCE)
set(PISA_ENABLE_TESTING OFF CACHE BOOL "" FORCE)
set(PISA_ENABLE_BENCHMARKING OFF CACHE BOOL "" FORCE)

find_package(Python REQUIRED COMPONENTS Interpreter Development.Module NumPy)
include_directories(${Python_NumPy_INCLUDE_DIRS})

# Guard x86-only SIMD headers/sources so PISA compiles on aarch64 as well.
foreach(_file
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/util/intrinsics.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/codec/VarIntG8IU.h"
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/MaskedVByte/src/varintdecode.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/MaskedVByte/src/varintencode.c")
  if(EXISTS "${_file}")
    file(READ "${_file}" _content)
    if(_content MATCHES "#include <x86intrin\\.h>")
      string(REGEX REPLACE
        "(#include <x86intrin\\.h>)"
        "#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__)\n\\1\n#endif"
        _content "${_content}")
      file(WRITE "${_file}" "${_content}")
      message(STATUS "ARM patch applied: ${_file}")
    endif()
  endif()
endforeach()

add_subdirectory(pisa)

Python_add_library(_pisathon MODULE src/pyterrier_pisa/_pisathon.cpp WITH_SOABI)
target_link_libraries(_pisathon PRIVATE pisa)

install(TARGETS _pisathon DESTINATION pyterrier_pisa)
