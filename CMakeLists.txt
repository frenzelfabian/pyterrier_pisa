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

add_subdirectory(pisa)

Python_add_library(_pisathon MODULE src/pyterrier_pisa/_pisathon.cpp WITH_SOABI)
target_link_libraries(_pisathon PRIVATE pisa)

install(TARGETS _pisathon DESTINATION pyterrier_pisa)
