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

# Guard x86-only SIMD headers so PISA compiles on aarch64 as well.
foreach(_file
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/util/intrinsics.hpp"
    "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/codec/VarIntG8IU.h")
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

# On non-x86, replace MaskedVByte sources with portable scalar VByte implementations.
# MaskedVByte uses SSE4.1/AVX2 SIMD that does not exist on aarch64.
if(NOT CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(amd64)|(AMD64)")
  set(_mb "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/MaskedVByte")
  if(IS_DIRECTORY "${_mb}")
    message(STATUS "Replacing MaskedVByte with portable scalar implementation for non-x86")
    file(WRITE "${_mb}/src/varintdecode.c" [=[
#include <stdint.h>
#include <stddef.h>
#include "../include/varintdecode.h"
size_t masked_vbyte_decode(const uint8_t *in, uint32_t *out, uint64_t length) {
    size_t pos = 0;
    for (uint64_t i = 0; i < length; i++) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        out[i] = v | ((uint32_t)in[pos++] << s);
    }
    return pos;
}
size_t masked_vbyte_decode_delta(const uint8_t *in, uint32_t *out, uint64_t length, uint32_t prev) {
    size_t pos = 0;
    for (uint64_t i = 0; i < length; i++) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        v |= (uint32_t)in[pos++] << s;
        prev += v; out[i] = prev;
    }
    return pos;
}
size_t masked_vbyte_decode_fromcompressedsize(const uint8_t *in, uint32_t *out, size_t inputsize) {
    size_t pos = 0, n = 0;
    while (pos < inputsize) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        out[n++] = v | ((uint32_t)in[pos++] << s);
    }
    return n;
}
size_t masked_vbyte_decode_fromcompressedsize_delta(const uint8_t *in, uint32_t *out, size_t inputsize, uint32_t prev) {
    size_t pos = 0, n = 0;
    while (pos < inputsize) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        v |= (uint32_t)in[pos++] << s;
        prev += v; out[n++] = prev;
    }
    return n;
}
uint32_t masked_vbyte_select_delta(const uint8_t *in, uint64_t length, uint32_t prev, size_t slot) {
    size_t pos = 0;
    for (uint64_t i = 0; i <= slot && i < length; i++) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        v |= (uint32_t)in[pos++] << s;
        prev += v;
    }
    return prev;
}
int masked_vbyte_search_delta(const uint8_t *in, uint64_t length, uint32_t prev, uint32_t key, uint32_t *presult) {
    size_t pos = 0;
    for (uint64_t i = 0; i < length; i++) {
        uint32_t v = 0; int s = 0;
        while (in[pos] & 0x80) { v |= (uint32_t)(in[pos++] & 0x7F) << s; s += 7; }
        v |= (uint32_t)in[pos++] << s;
        prev += v;
        if (prev >= key) { *presult = prev; return (int)i; }
    }
    return -1;
}
]=])
    file(WRITE "${_mb}/src/varintencode.c" [=[
#include <stdint.h>
#include <stddef.h>
#include "../include/varintencode.h"
size_t vbyte_encode(uint32_t *in, size_t length, uint8_t *bout) {
    size_t pos = 0;
    for (size_t i = 0; i < length; i++) {
        uint32_t v = in[i];
        while (v > 0x7F) { bout[pos++] = (uint8_t)((v & 0x7F) | 0x80); v >>= 7; }
        bout[pos++] = (uint8_t)v;
    }
    return pos;
}
size_t vbyte_encode_delta(uint32_t *in, size_t length, uint8_t *bout, uint32_t prev) {
    size_t pos = 0;
    for (size_t i = 0; i < length; i++) {
        uint32_t v = in[i] - prev; prev = in[i];
        while (v > 0x7F) { bout[pos++] = (uint8_t)((v & 0x7F) | 0x80); v >>= 7; }
        bout[pos++] = (uint8_t)v;
    }
    return pos;
}
]=])
  endif()
endif()

# On non-x86, replace simdcomp C sources with empty stubs and simdbp.cpp with
# an interpolative fallback. simdcomp uses SSE2 (__m128i) types that don't exist on aarch64.
if(NOT CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(amd64)|(AMD64)")
  set(_sc "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/simdcomp")
  if(IS_DIRECTORY "${_sc}")
    message(STATUS "Replacing simdcomp with empty stubs for non-x86")
    file(WRITE "${_sc}/src/simdbitpacking.c" "/* ARM stub: no SSE2 */\n")
    file(WRITE "${_sc}/src/simdcomputil.c" "/* ARM stub: no SSE2 */\n")
  endif()
  set(_simdbp "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/simdbp.cpp")
  if(EXISTS "${_simdbp}")
    message(STATUS "Replacing simdbp.cpp with interpolative fallback for non-x86")
    file(WRITE "${_simdbp}" [=[
#include "codec/simdbp.hpp"
#include "codec/block_codecs.hpp"
namespace pisa {
void SimdBpBlockCodec::encode(
    uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out
) const {
    auto* src = const_cast<uint32_t*>(in);
    interpolative_block::encode(src, sum_of_values, n, out);
}
uint8_t const*
SimdBpBlockCodec::decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const {
    return interpolative_block::decode(in, out, sum_of_values, n);
}
}  // namespace pisa
]=])
  endif()
endif()

add_subdirectory(pisa)

Python_add_library(_pisathon MODULE src/pyterrier_pisa/_pisathon.cpp WITH_SOABI)
target_link_libraries(_pisathon PRIVATE pisa)

install(TARGETS _pisathon DESTINATION pyterrier_pisa)
