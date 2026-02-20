cmake_minimum_required(VERSION 3.10)

project(whisper_flutter_library LANGUAGES C CXX)
set(CMAKE_CXX_STANDARD 20)

# Optimization flags
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -O3 -fdata-sections -ffunction-sections")
set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -fdata-sections -ffunction-sections")

# POSIX
add_compile_definitions(_XOPEN_SOURCE=600)
add_compile_definitions(_GNU_SOURCE)

# Version defines (from src/include/whisper-version.h)
add_compile_definitions(GGML_VERSION="1.8.3")
add_compile_definitions(GGML_COMMIT="7aa8818")
add_compile_definitions(WHISPER_VERSION="1.8.3")

# Include directories — corrected for nested whisper.cpp v1.8.x layout
include_directories(src)                    # whisper.h at top level
include_directories(src/include)            # public whisper headers
include_directories(src/src)                # whisper-arch.h
include_directories(src/ggml/include)       # ggml.h, ggml-backend.h, etc.
include_directories(src/ggml/src)           # ggml-impl.h, ggml-common.h, etc.
include_directories(src/ggml/src/ggml-cpu)  # ggml-cpu-impl.h, ops.h, etc.

# ---- GGML base sources ----
file(GLOB GGML_BASE_SRC
    src/ggml/src/ggml.c
    src/ggml/src/ggml.cpp
    src/ggml/src/ggml-alloc.c
    src/ggml/src/ggml-backend.cpp
    src/ggml/src/ggml-backend-reg.cpp
    src/ggml/src/ggml-opt.cpp
    src/ggml/src/ggml-threading.cpp
    src/ggml/src/ggml-quants.c
    src/ggml/src/gguf.cpp
)

# ---- GGML-CPU sources ----
file(GLOB GGML_CPU_SRC
    src/ggml/src/ggml-cpu/ggml-cpu.c
    src/ggml/src/ggml-cpu/ggml-cpu.cpp
    src/ggml/src/ggml-cpu/ops.cpp
    src/ggml/src/ggml-cpu/quants.c
    src/ggml/src/ggml-cpu/binary-ops.cpp
    src/ggml/src/ggml-cpu/unary-ops.cpp
    src/ggml/src/ggml-cpu/repack.cpp
    src/ggml/src/ggml-cpu/hbm.cpp
    src/ggml/src/ggml-cpu/traits.cpp
    src/ggml/src/ggml-cpu/vec.cpp
    src/ggml/src/ggml-cpu/amx/amx.cpp
    src/ggml/src/ggml-cpu/amx/mmq.cpp
)

# ARM-specific sources (Android is always ARM)
file(GLOB GGML_CPU_ARM_SRC
    src/ggml/src/ggml-cpu/arch/arm/cpu-feats.cpp
    src/ggml/src/ggml-cpu/arch/arm/quants.c
    src/ggml/src/ggml-cpu/arch/arm/repack.cpp
)

# ---- Whisper source ----
set(WHISPER_SRC
    src/src/whisper.cpp
)

add_library(whisper
    ${GGML_BASE_SRC}
    ${GGML_CPU_SRC}
    ${GGML_CPU_ARM_SRC}
    ${WHISPER_SRC}
)

target_compile_features(whisper PRIVATE c_std_11 cxx_std_17)

if (${ANDROID_ABI} STREQUAL "arm64-v8a")
    target_compile_options(whisper PRIVATE -march=armv8.2-a+fp16+dotprod)
elseif (${ANDROID_ABI} STREQUAL "armeabi-v7a")
    target_compile_options(whisper PRIVATE -mfpu=neon-vfpv4)
endif ()

find_library(LOG_LIB log)
target_link_libraries(whisper PRIVATE ${LOG_LIB})

# Main bridge library
add_library(whisper_flutter SHARED main.cpp)
target_link_libraries(whisper_flutter PRIVATE whisper)
target_compile_definitions(whisper_flutter PUBLIC DART_SHARED_LIB)

set_target_properties(whisper_flutter PROPERTIES
  PUBLIC_HEADER src/whisper.h
  OUTPUT_NAME "whisper"
)
