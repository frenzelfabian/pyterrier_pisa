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

  # FastPFor uses SSE2/AVX2 intrinsics (immintrin.h) that don't exist on aarch64.
  # Replace the three compiled bitpacking source files with empty stubs so the
  # FastPFor STATIC library target compiles, then replace the PISA codec files
  # that use FastPFor types with interpolative fallbacks.
  set(_fp "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/FastPFor")
  if(IS_DIRECTORY "${_fp}")
    message(STATUS "Replacing FastPFor compiled sources with empty stubs for non-x86")
    foreach(_f "bitpackingaligned" "bitpacking" "bitpackingunaligned")
      file(WRITE "${_fp}/src/${_f}.cpp" "/* ARM stub: no SSE2/AVX2 */\n")
    endforeach()
  endif()
  set(_optpfor_hpp "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/codec/optpfor.hpp")
  if(EXISTS "${_optpfor_hpp}")
    message(STATUS "Replacing optpfor.hpp with ARM-safe stub for non-x86")
    file(WRITE "${_optpfor_hpp}" [=[
#pragma once
#include <vector>
#include "codec/block_codec.hpp"
namespace pisa {
class OptPForBlockCodec: public BlockCodec {
    static const uint64_t m_block_size = 128;
  public:
    constexpr static std::string_view name = "block_optpfor";
    virtual ~OptPForBlockCodec() = default;
    void encode(uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out) const override;
    uint8_t const* decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const override;
    auto block_size() const noexcept -> std::size_t override { return m_block_size; }
    auto get_name() const noexcept -> std::string_view override { return name; }
};
}  // namespace pisa
]=])
  endif()
  foreach(_codec "optpfor;OptPForBlockCodec;optpfor" "simple8b;Simple8bBlockCodec;simple8b" "simple16;Simple16BlockCodec;simple16")
    list(GET _codec 0 _name)
    list(GET _codec 1 _classname)
    list(GET _codec 2 _hdr)
    set(_f "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/${_name}.cpp")
    if(EXISTS "${_f}")
      message(STATUS "Replacing ${_name}.cpp with interpolative fallback for non-x86")
      file(WRITE "${_f}" "#include \"codec/${_hdr}.hpp\"\n#include \"codec/block_codecs.hpp\"\nnamespace pisa {\nvoid ${_classname}::encode(uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out) const {\n    auto* src = const_cast<uint32_t*>(in);\n    interpolative_block::encode(src, sum_of_values, n, out);\n}\nuint8_t const* ${_classname}::decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const {\n    return interpolative_block::decode(in, out, sum_of_values, n);\n}\n}  // namespace pisa\n")
    endif()
  endforeach()

  # QMX uses SSE4.1/SSE4.2 intrinsics. Replace its CMakeLists.txt with a no-op
  # INTERFACE library so the target exists without compiling any SIMD sources,
  # and replace qmx.cpp with an interpolative fallback.
  set(_qmx_dir "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/QMX")
  if(IS_DIRECTORY "${_qmx_dir}")
    message(STATUS "Replacing QMX with empty INTERFACE library for non-x86")
    file(WRITE "${_qmx_dir}/CMakeLists.txt" "add_library(QMX INTERFACE)\n")
  endif()
  set(_qmx_cpp "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/qmx.cpp")
  if(EXISTS "${_qmx_cpp}")
    message(STATUS "Replacing qmx.cpp with interpolative fallback for non-x86")
    file(WRITE "${_qmx_cpp}" "#include \"codec/qmx.hpp\"\n#include \"codec/block_codecs.hpp\"\nnamespace pisa {\nvoid QmxBlockCodec::encode(uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out) const {\n    auto* src = const_cast<uint32_t*>(in);\n    interpolative_block::encode(src, sum_of_values, n, out);\n}\nuint8_t const* QmxBlockCodec::decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const {\n    return interpolative_block::decode(in, out, sum_of_values, n);\n}\n}  // namespace pisa\n")
  endif()
endif()

add_subdirectory(pisa)

Python_add_library(_pisathon MODULE src/pyterrier_pisa/_pisathon.cpp WITH_SOABI)
target_link_libraries(_pisathon PRIVATE pisa)

install(TARGETS _pisathon DESTINATION pyterrier_pisa)
