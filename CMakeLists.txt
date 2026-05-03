cmake_minimum_required(VERSION 3.15)

project(${SKBUILD_PROJECT_NAME} LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS ON)

# Detect target architecture once and set portable flags. The submodule's own
# CMakeLists files append `-march=native` later (we patch that out below), so
# these flags determine what actually gets used.
if(CMAKE_SYSTEM_PROCESSOR MATCHES "(x86_64)|(amd64)|(AMD64)")
    set(_PISA_ARCH_X86 ON)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mtune=generic -march=x86-64 -msse4.2 -mavx -mavx2 -mfma -mbmi -mrdseed -mpopcnt")
elseif(CMAKE_SYSTEM_PROCESSOR MATCHES "(aarch64)|(arm64)|(ARM64)")
    set(_PISA_ARCH_X86 OFF)
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -mtune=generic")
else()
    set(_PISA_ARCH_X86 OFF)
endif()

# Speed up the build by removing CLI tools, tests, and benchmarks
set(PISA_BUILD_TOOLS OFF CACHE BOOL "" FORCE)
set(PISA_ENABLE_TESTING OFF CACHE BOOL "" FORCE)
set(PISA_ENABLE_BENCHMARKING OFF CACHE BOOL "" FORCE)

find_package(Python REQUIRED COMPONENTS Interpreter Development.Module NumPy)
include_directories(${Python_NumPy_INCLUDE_DIRS})

# ---------------------------------------------------------------------------
# Idempotent patching helpers
#
# The submodule pisa/ ships sources that assume x86 (SSE/AVX intrinsics).
# To make builds work on aarch64 we patch a few headers/sources at configure
# time. These patches MUST be idempotent: CMake reconfigures repeatedly
# during builds, and a non-idempotent patch ends up nesting guards or
# bumping file mtimes on every run, which triggers spurious rebuilds and,
# in some cases, slowly corrupts the file.
# ---------------------------------------------------------------------------

# Write `_content` to `_path` only if the existing file differs. Avoids
# bumping mtimes on every reconfigure.
function(_pisa_write_if_changed _path _content)
    set(_existing "")
    if(EXISTS "${_path}")
        file(READ "${_path}" _existing)
    endif()
    if(NOT _existing STREQUAL "${_content}")
        file(WRITE "${_path}" "${_content}")
        message(STATUS "pyterrier_pisa: wrote ${_path}")
    endif()
endfunction()

# Wrap `#include <x86intrin.h>` in an architecture guard. Idempotent: looks
# at the bytes preceding the include and skips if `__x86_64__` already guards
# it.
function(_pisa_guard_x86intrin _file)
    if(NOT EXISTS "${_file}")
        return()
    endif()
    file(READ "${_file}" _content)
    string(FIND "${_content}" "#include <x86intrin.h>" _inc_pos)
    if(_inc_pos LESS 0)
        return()
    endif()
    string(SUBSTRING "${_content}" 0 ${_inc_pos} _before)
    if(_before MATCHES "defined\\(__x86_64__\\)")
        # Already guarded — nothing to do.
        return()
    endif()
    string(REGEX REPLACE
        "#include <x86intrin\\.h>"
        "#if defined(__x86_64__) || defined(_M_X64) || defined(__i386__)\n#include <x86intrin.h>\n#endif"
        _content "${_content}")
    file(WRITE "${_file}" "${_content}")
    message(STATUS "pyterrier_pisa: ARM guard applied to ${_file}")
endfunction()

# Drop unconditional `-march=native` lines from the submodule CMakeLists files.
# The parent already sets portable arch flags above; `-march=native` overrides
# them and breaks reproducible/portable builds. Idempotent: if no match found,
# nothing is rewritten.
function(_pisa_strip_march_native _file)
    if(NOT EXISTS "${_file}")
        return()
    endif()
    file(READ "${_file}" _content)
    string(REGEX REPLACE
        "set\\(CMAKE_(C|CXX)_FLAGS \"\\\$\\{CMAKE_(C|CXX)_FLAGS\\} -march=native\"\\)"
        "# (-march=native removed; arch flags set by parent CMakeLists)"
        _new "${_content}")
    if(NOT _new STREQUAL "${_content}")
        file(WRITE "${_file}" "${_new}")
        message(STATUS "pyterrier_pisa: stripped -march=native from ${_file}")
    endif()
endfunction()

# Apply x86intrin guards.
_pisa_guard_x86intrin("${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/util/intrinsics.hpp")
_pisa_guard_x86intrin("${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/codec/VarIntG8IU.h")

# Strip redundant -march=native from submodule CMakeLists.
_pisa_strip_march_native("${CMAKE_CURRENT_SOURCE_DIR}/pisa/CMakeLists.txt")
_pisa_strip_march_native("${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/CMakeLists.txt")

# ---------------------------------------------------------------------------
# Architecture-specific source replacements (non-x86 only)
#
# On aarch64, several SIMD libraries don't compile because they rely on x86
# intrinsics (SSE/AVX). For each, we replace the offending sources with a
# portable fallback (or empty stub if the symbols aren't actually called).
# All writes go through _pisa_write_if_changed for idempotency.
# ---------------------------------------------------------------------------
if(NOT _PISA_ARCH_X86)
  # MaskedVByte: SSE4.1/AVX2 → portable scalar VByte.
  set(_mb "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/MaskedVByte")
  if(IS_DIRECTORY "${_mb}")
    _pisa_write_if_changed("${_mb}/src/varintdecode.c" [=[
/* PISA_ARM_STUB: portable scalar replacement for MaskedVByte SSE/AVX code. */
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
    _pisa_write_if_changed("${_mb}/src/varintencode.c" [=[
/* PISA_ARM_STUB: portable scalar replacement for MaskedVByte SSE/AVX code. */
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

  # simdcomp: SSE2 (__m128i) → empty stubs (symbols aren't reached on ARM
  # because SimdBpBlockCodec is replaced below with an interpolative fallback).
  set(_sc "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/simdcomp")
  if(IS_DIRECTORY "${_sc}")
    _pisa_write_if_changed("${_sc}/src/simdbitpacking.c" "/* PISA_ARM_STUB: no SSE2 */\n")
    _pisa_write_if_changed("${_sc}/src/simdcomputil.c" "/* PISA_ARM_STUB: no SSE2 */\n")
  endif()

  # FastPFor: SSE2/AVX2 (immintrin.h) → empty stubs for the bitpacking sources.
  set(_fp "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/FastPFor")
  if(IS_DIRECTORY "${_fp}")
    foreach(_f "bitpackingaligned" "bitpacking" "bitpackingunaligned" "simdbitpacking")
      _pisa_write_if_changed("${_fp}/src/${_f}.cpp" "/* PISA_ARM_STUB: no SSE2/AVX2 */\n")
    endforeach()
  endif()

  # PISA codec sources that depend on the SIMD libs above. Replace each with
  # an interpolative fallback so the symbols still link.
  _pisa_write_if_changed(
      "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/simdbp.cpp"
      [=[
/* PISA_ARM_STUB: interpolative fallback for SimdBpBlockCodec. */
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

  # OptPFor's header pulls in FastPFor types. Replace with an ARM-safe stub
  # whose .cpp uses an interpolative fallback.
  _pisa_write_if_changed(
      "${CMAKE_CURRENT_SOURCE_DIR}/pisa/include/pisa/codec/optpfor.hpp"
      [=[
/* PISA_ARM_STUB: header without FastPFor SIMD dependency. */
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

  # Codec .cpp files that compile against the now-stubbed SIMD libs. Each is
  # replaced with an interpolative fallback. Format: "name:class:header" —
  # using ':' as the inner separator because CMake list semantics flatten
  # nested ';' lists.
  set(_codec_specs
      "optpfor:OptPForBlockCodec:optpfor"
      "simple8b:Simple8bBlockCodec:simple8b"
      "simple16:Simple16BlockCodec:simple16"
  )
  foreach(_spec IN LISTS _codec_specs)
    string(REGEX MATCHALL "[^:]+" _parts "${_spec}")
    list(GET _parts 0 _name)
    list(GET _parts 1 _classname)
    list(GET _parts 2 _hdr)
    set(_target "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/${_name}.cpp")
    set(_codec_body
"/* PISA_ARM_STUB: interpolative fallback for ${_classname}. */
#include \"codec/${_hdr}.hpp\"
#include \"codec/block_codecs.hpp\"
namespace pisa {
void ${_classname}::encode(uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out) const {
    auto* src = const_cast<uint32_t*>(in);
    interpolative_block::encode(src, sum_of_values, n, out);
}
uint8_t const* ${_classname}::decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const {
    return interpolative_block::decode(in, out, sum_of_values, n);
}
}  // namespace pisa
")
    _pisa_write_if_changed("${_target}" "${_codec_body}")
  endforeach()

  # QMX uses SSE4.1/SSE4.2 intrinsics. Replace its CMakeLists with a no-op
  # INTERFACE library so the target exists without compiling SIMD sources,
  # then replace qmx.cpp with an interpolative fallback.
  set(_qmx_dir "${CMAKE_CURRENT_SOURCE_DIR}/pisa/external/QMX")
  if(IS_DIRECTORY "${_qmx_dir}")
    _pisa_write_if_changed("${_qmx_dir}/CMakeLists.txt"
        "# PISA_ARM_STUB: no SSE4 on aarch64\nadd_library(QMX INTERFACE)\n")
  endif()
  _pisa_write_if_changed(
      "${CMAKE_CURRENT_SOURCE_DIR}/pisa/src/codec/qmx.cpp"
      [=[
/* PISA_ARM_STUB: interpolative fallback for QmxBlockCodec. */
#include "codec/qmx.hpp"
#include "codec/block_codecs.hpp"
namespace pisa {
void QmxBlockCodec::encode(uint32_t const* in, uint32_t sum_of_values, size_t n, std::vector<uint8_t>& out) const {
    auto* src = const_cast<uint32_t*>(in);
    interpolative_block::encode(src, sum_of_values, n, out);
}
uint8_t const* QmxBlockCodec::decode(uint8_t const* in, uint32_t* out, uint32_t sum_of_values, size_t n) const {
    return interpolative_block::decode(in, out, sum_of_values, n);
}
}  // namespace pisa
]=])
endif()

add_subdirectory(pisa)

Python_add_library(_pisathon MODULE src/pyterrier_pisa/_pisathon.cpp WITH_SOABI)
target_link_libraries(_pisathon PRIVATE pisa)

install(TARGETS _pisathon DESTINATION pyterrier_pisa)
